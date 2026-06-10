# Contribution #1: Add WordTTSService support for AWS PollyTTSService

**Contribution Number:** 1
**Student:** Eduardo Perez
**Issue:** https://github.com/pipecat-ai/pipecat/issues/1015
**Status:** Phase II Complete

## Why I Chose This Issue

This issue combines real-time voice AI (Pipecat's core domain) with AWS services I
want more hands-on experience with. It is labeled `help wanted`, the maintainers
explicitly welcomed a community PR, and a maintainer (markbackman) approved my
proposed scope, which gave me confidence it is well-scoped for a first contribution.
The work touches an interesting architectural challenge: the existing word-timestamp
implementations are mostly websocket-based, while Polly is HTTP-based and requires a
separate API call for timing data.

## Understanding the Issue

### Problem Description

`AWSPollyTTSService` (Pipecat's Amazon Polly TTS integration) generates speech audio
but provides no per-word timing information. Other TTS services like
`ElevenLabsTTSService` emit one timestamped `TTSTextFrame` per word, which
downstream consumers use for captions, RTVI bot-output events, and accurate
assistant context tracking. Amazon Polly supports word timing via its SpeechMarks
feature, but the Pipecat service never requests or uses it.

Note: the issue title references `WordTTSService` and `PollyTTSService`. Both names
are outdated. `WordTTSService` was deprecated (word-timestamp support now lives in
the base `TTSService`), and the service class is currently named
`AWSPollyTTSService` in `src/pipecat/services/aws/tts.py`.

### Expected Behavior

When synthesizing a sentence, the service should emit one `TTSTextFrame` per word,
each with a presentation timestamp (`pts`) aligned to when that word is spoken in
the audio, the same behavior ElevenLabs and other timestamp-capable services
provide.

### Current Behavior

The service emits the audio frames plus a single whole-sentence `TTSTextFrame` with
`pts=None`. No per-word timing exists anywhere in the output.

### Affected Components

- `src/pipecat/services/aws/tts.py` — `AWSPollyTTSService` (the only file needing
  code changes)
- `src/pipecat/services/tts_service.py` — base `TTSService` word-timestamp
  machinery (`add_word_timestamps()`), used but not modified
- Reference implementation: `src/pipecat/services/inworld/tts.py`
  (`InworldHttpTTSService`)

## Reproduction Process

### Environment Setup

Windows 11, Python 3.11.9, uv 0.11.8. Pipecat's documented dev setup is:

```
uv sync --group dev --all-extras --no-extra gstreamer --no-extra local
```

Two issues encountered and solved:

1. **`daily-python` has no Windows wheels.** The sync failed with
   "can't be installed because it doesn't have a source distribution or wheel for
   the current platform" (only manylinux/macOS wheels exist).
2. **Excluding the `daily` extra was not enough.** Adding `--no-extra daily` still
   failed because other extras (e.g. `tavus`, `heygen`) transitively depend on
   `pipecat-ai[daily]`, pulling `daily-python` back in.

**Solution:** skip `--all-extras` and install only what this contribution needs:

```
uv sync --group dev --extra aws
uv run pre-commit install
```

This installs the dev toolchain (pytest, ruff, pre-commit) plus the AWS
dependencies (aiobotocore). Verified with a successful import of
`AWSPollyTTSService`.

AWS credentials were configured locally via `aws configure` (IAM access key,
region `us-east-1`) and verified with `aws sts get-caller-identity`.

### Steps to Reproduce

1. Clone the fork and check out the working branch:
   `git clone https://github.com/perez-eduardo/pipecat.git` and
   `git checkout add-aws-tts-word-timestamps-1015`
2. Set up the environment: `uv sync --group dev --extra aws`
3. Configure AWS credentials with Polly access (`aws configure`, region
   `us-east-1`)
4. Confirm Polly SpeechMarks works at the API level:
   ```
   aws polly synthesize-speech --output-format json --speech-mark-types '[\"word\"]' --text "Hello world from Pipecat" --voice-id Joanna marks.json
   ```
   Returns one JSON object per word with millisecond timing:
   `{"time":6,"type":"word","start":0,"end":5,"value":"Hello"}` — proving the data
   source for the fix exists.
5. Run the reproduction script (in repo root, not committed):
   `uv run python repro_issue_1015_polly_word_timestamps.py`
   The script pushes a 9-word `TTSSpeakFrame` through `AWSPollyTTSService` using
   Pipecat's official `run_test()` harness and inspects every downstream frame.
6. **Expected (per issue):** 9 timestamped `TTSTextFrame`s (one per word).
   **Actual:** 1 whole-sentence `TTSTextFrame` with `pts=None`, 6
   `TTSAudioRawFrame`s, and zero per-word timing:
   ```
   Input text word count:        9
   TTSTextFrame count:           1
   TTSTextFrames with pts set:   0
   RESULT: NO per-word timestamped TTSTextFrames were produced.
   ```
7. Reproduced 3 times with identical results (consistent, not a fluke).

Additional code-level evidence: `Select-String` over `src/pipecat/services/aws/tts.py`
finds **zero** references to `word_timestamp`, `speech_mark`, or `SpeechMark`, while
`src/pipecat/services/elevenlabs/tts.py` calls `add_word_timestamps()` at lines 903
and 1455.

### Reproduction Evidence

- Working branch: https://github.com/perez-eduardo/pipecat/tree/add-aws-tts-word-timestamps-1015
- Reproduction script + full output: `repro_issue_1015_polly_word_timestamps.py`
  (kept out of the pipecat fork; script and captured run log stored alongside this
  README)
- My findings: the gap is confirmed at both the code level (no SpeechMarks usage
  anywhere in the AWS TTS service) and runtime level (no per-word frames emitted).
  Polly's SpeechMarks API verifiably returns the needed word timing data, but it
  requires a second `synthesize_speech` call (`OutputFormat="json"`) because PCM
  audio and JSON marks cannot be returned in a single response.

## Solution Approach

### Analysis

Root cause: `AWSPollyTTSService.run_tts()` makes a single `synthesize_speech` call
with `OutputFormat="pcm"` and yields only audio frames. It never calls the base
class's `add_word_timestamps()`, and its `super().__init__()` doesn't pass
`push_text_frames=False`, so the base class pushes the whole sentence as one
untimed text frame. The base `TTSService` already contains all the timestamp
machinery (PTS baseline set on the first audio frame of a context, per-word
`TTSTextFrame` emission, ordering via the per-context audio queue); the service
simply never feeds it.

### Proposed Solution

Add a second, concurrent Polly `synthesize_speech` call requesting word SpeechMarks
(`OutputFormat="json"`, `SpeechMarkTypes=["word"]`), parse the newline-delimited
JSON response into `(word, start_seconds)` pairs with cumulative timing across the
turn, and feed them to the base class via `add_word_timestamps()`, following the
established `InworldHttpTTSService` pattern for HTTP-based TTS services.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Polly TTS emits audio but a single untimed whole-sentence text
frame; downstream word-timing consumers get nothing. Polly exposes word timing via
SpeechMarks but only through a separate API call, which the service never makes.

**Match:** `InworldHttpTTSService` (`src/pipecat/services/inworld/tts.py`) is the
closest existing pattern: an HTTP-based TTS service that passes
`push_text_frames=False`, keeps a `_cumulative_time` accumulator (reset on
`InterruptionFrame`/`TTSStoppedFrame` via a `push_frame` override), converts API
timing to cumulative seconds, calls `add_word_timestamps()`, and falls back to a
whole-text `TTSTextFrame` when no timing data returns. Notably, neither
`GoogleHttpTTSService` nor OpenAI TTS implement word timestamps, so Inworld is the
reference.

**Plan:** (all in `src/pipecat/services/aws/tts.py`)
1. `__init__`: pass `push_text_frames=False` to `super().__init__()`; initialize
   `self._cumulative_time = 0.0`.
2. Add `push_frame` override resetting `_cumulative_time` on `InterruptionFrame` /
   `TTSStoppedFrame`.
3. Add `_fetch_speech_marks` helper: second `synthesize_speech` call with
   `OutputFormat="json"`, `SpeechMarkTypes=["word"]`; parse NDJSON into
   `[(value, cumulative_time + time_ms / 1000)]`.
4. `run_tts`: run audio and marks calls concurrently (`asyncio.gather`) to avoid
   doubling TTFB; call `add_word_timestamps(word_times, context_id)`; advance
   `_cumulative_time` by utterance duration computed from PCM payload size (Polly
   marks have no end times); keep a no-marks fallback that pushes one whole-text
   `TTSTextFrame` so assistant context is still recorded.
5. Add changelog fragment `changelog/<PR_number>.added.md` per CONTRIBUTING.md.

**Implement:** Working branch:
https://github.com/perez-eduardo/pipecat/tree/add-aws-tts-word-timestamps-1015

**Review:** Self-review against CONTRIBUTING.md before the PR: Google-style
docstrings, Ruff (line length 100) via installed pre-commit hooks, changelog
fragment, follow existing TTS provider patterns and base-class methods (explicit
maintainer guidance from markbackman).

**Evaluate:** Mocked unit tests (no live AWS in CI), reproduction script re-run
showing per-word frames, manual audio/word alignment check in the Pipecat Prebuilt
UI (maintainer's explicit request), full `uv run pytest` and ruff checks.

## Testing Strategy

### Unit Tests

(All using `run_test()` with mocked Polly responses, the scope the maintainer
approved, so no live AWS credentials are needed in CI.)

- Test case 1: a single sentence produces one `TTSTextFrame` per word with
  monotonically increasing `pts`.
- Test case 2: multiple sentences within one turn produce cumulative timestamps
  (second sentence's words offset by the first sentence's audio duration).
- Test case 3: SpeechMarks returning empty/no data triggers the fallback whole-text
  `TTSTextFrame` (assistant context still recorded).
- Test case 4: interruption resets `_cumulative_time` so the next turn's
  timestamps start from zero.

### Integration Tests

- Live (local, not CI): reproduction script re-run on the fixed branch flips the
  gap analysis to per-word timestamped frames ≥ word count.
- Full repo test suite (`uv run pytest`) passes with no regressions.

### Manual Testing

- Pipecat Prebuilt UI session with live AWS credentials to visually confirm
  audio/word alignment (the maintainer's requested verification).
- Result: pending Phase III implementation.

## Implementation Notes

### Week 2 Progress

- Set up Windows dev environment with uv; diagnosed and worked around
  `daily-python`'s missing Windows wheels (targeted `--extra aws` install instead
  of `--all-extras`).
- Created and pushed working branch `add-aws-tts-word-timestamps-1015`.
- Verified Polly SpeechMarks at the AWS API level (CloudShell and local CLI).
- Built a reproduction script using Pipecat's `run_test()` harness; reproduced the
  gap 3 times consistently.
- Traced the root cause and reference pattern using Claude Code (read-only
  analysis); discovered the issue's class names are outdated and that
  `InworldHttpTTSService`, not ElevenLabs, is the right HTTP-based reference.
- Wrote the UMPIRE implementation plan.
- Decisions made: branch named for current reality (`aws-tts`) rather than the
  issue's outdated `PollyTTSService` name; mocked tests for CI per
  maintainer-approved scope; concurrent SpeechMarks call to protect TTFB.

### Code Changes

- Files modified: (Phase III) `src/pipecat/services/aws/tts.py`,
  `changelog/<PR>.added.md`, new test file
- Key commits: pending Phase III.
- Approach decisions: follow `InworldHttpTTSService` pattern; use base `TTSService`
  machinery only (no base-class changes); `asyncio.gather` for the dual Polly
  calls.

## Pull Request

**PR Link:** Not yet submitted (Phase IV).
**PR Description:** Will be drafted in Phase IV, adapted from the Solution Approach
and Testing Strategy sections above.
**Maintainer Feedback:**

- 2026-06 (pre-work, on issue #1015): markbackman approved the proposed scope;
  asked to confirm audio/word alignment via Pipecat Prebuilt and to follow existing
  TTS provider patterns and base TTSService class methods.

**Status:** Phase II Complete — ready to begin Phase III (Build)

## Learnings & Reflections

### Technical Skills Gained

Managing Python dependencies with uv, including extras and the realities of
platform-specific wheels. Pipecat's frame-based pipeline architecture, especially
how the base `TTSService` manages audio contexts, PTS baselines, and word
timestamps so individual services only need to supply timing data. The Amazon Polly
SpeechMarks API and its constraint that audio and timing require separate requests.
Using Pipecat's `run_test()` harness as a precise reproduction instrument rather
than relying on manual observation. Using Claude Code for read-only codebase
analysis to find root causes and reference patterns before writing any code.

### Challenges Overcome

The documented dev setup failed on Windows because `daily-python` ships no Windows
wheels, and excluding its extra wasn't enough since other extras pull it back in
transitively; solving this required understanding how uv resolves extras and
switching to a targeted install. The issue text itself was a moving target: both
class names it references (`WordTTSService`, `PollyTTSService`) no longer exist, so
I had to verify the current state of the code instead of trusting the issue or even
the changelog. Finally, I accidentally exposed an AWS access key while sharing
terminal output and had to immediately deactivate, delete, and rotate it; a
concrete lesson in treating any pasted secret as compromised and never sharing
`aws configure` output.

### What I'd Do Differently Next Time

Interrogate the live codebase before forming a plan, not after. My first
assumptions (the class name, and that Google/OpenAI HTTP TTS would be the reference
pattern) were both wrong, and verifying against the actual code corrected the plan
early instead of mid-implementation. I'd also handle credentials more carefully
from the start: create a narrowly-scoped IAM user for the project and sanitize
terminal output before sharing it anywhere.

## Resources Used

- Issue: https://github.com/pipecat-ai/pipecat/issues/1015
- Amazon Polly SpeechMarks: https://docs.aws.amazon.com/polly/latest/dg/speechmarks.html
- Pipecat CONTRIBUTING.md (changelog fragments, docstring conventions)
- Pipecat AGENTS.md (dev commands, architecture, test utilities)
- Word Timing reference: https://docs.pipecat.ai/server/services/tts/elevenlabs#word-timing
- Reference implementation: `src/pipecat/services/inworld/tts.py`
