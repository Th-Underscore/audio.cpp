# BreezeTTS Real-Time (Sub-Chunk) Streaming

Status: proposal. Work branch: `realtime-breeze-stream` (worktree
`/home/theunderscore/build/audio-realtime`, based on `sm70-attention-fallback`;
rebase onto `dev` before upstreaming). **Do not push to
`sm70-attention-fallback`** — PR #423 is open against it.

## Goal

First audio in ~1-2 s and continuous SSE `speech.audio.delta` flow while
generating, instead of one event per text chunk (~7 s to first audio for a
160-char chunk on V100-eager). No server changes: smaller, more frequent
events flow through the existing `/v1/audio/speech` SSE path automatically.

## Background

- `eb2cbd3` ("Add BreezeTTS streaming mode") is chunk-granular pull
  streaming: `start_stream` splits text, each `next_stream_event` runs the
  monolithic `BreezeGeneratorRuntime::generate()` for one chunk
  (`src/models/breeze_tts/session.cpp:279`).
- The AR loop itself is already per-frame (`generator.cpp:859`): backbone
  cond+uncond with CFG, `generate_frame` via the depth decoder, append
  `num_codebooks` (16) codes per frame, break on EOS. Natural callback
  insertion point after each frame.
- The blocker is `speech_decoder->decode()`, which takes the whole code
  sequence (24 kHz output). No incremental codec path exists.

## Design

1. **Resumable generator stepper.** Refactor `Impl::generate` so AR state
   (backbone decode states, sampler RNG/history, accumulated `codes`)
   persists across calls. New method, e.g.
   `generate_prefix_frames(request, state, max_new_frames, frame_callback)`,
   invoking the callback every M frames with the code prefix so far.
   Monolithic `generate()` is kept (implemented as: step to EOS, then
   decode) so offline behavior is byte-identical.
2. **Prefix decode with lookahead margin.** On each callback, build
   `BreezeSpeechCodes` from `codes[0..n]`, run the existing whole-sequence
   `decode`, and emit only samples up to frame `n-K` (`K` =
   `stream_lookahead_margin`, default TBD ~8-16 frames). The margin absorbs
   convolutional boundary artifacts; the tail is re-decoded on later
   callbacks and fully emitted at EOS. Decode-prefix-each-time is O(n^2)
   total but AR dominates (2 backbone forwards + depth per frame), so it
   stays real-time.
3. **Session pull integration.** `start_stream` creates the stepper state
   per chunk (or per request); `next_stream_event` pulls up to M frames
   (`stream_frames_per_event`, default TBD ~25-50) and returns them as one
   `speech.audio.delta`. Seam artifacts are tuned by ear via K and an
   optional short crossfade at emission boundaries.

New request/session options (all with offline-safe defaults, spec-backed
where the contract requires): `stream_frames_per_event`,
`stream_lookahead_margin`. If the contract rejects unknown options in
offline mode, gate them to streaming sessions only.

## Validation

- Prefix-vs-full decode parity: final concatenated stream must match
  monolithic `generate()` output within a small tolerance (exact match
  expected except the margin region; assert on the non-margin prefix).
- Seam audibility: ear-check + spectral discontinuity metric at emission
  boundaries for several K values.
- Timing: time-to-first-audio and inter-event gaps on V100 (note: test GPU
  underclocked to 765 MHz — wall times understate full-clock performance);
  RTF under repeated requests; peak VRAM.
- Regression: `ctest` green, Fish/Higgs untouched, offline Breeze output
  unchanged.
- Server: multi-event SSE arrival timestamps (existing technique:
  per-event `t+` reader) proving progressive delivery.

## Validation results (2026-09-02, V100 765 MHz underclock, eager fallback)

- Offline parity: stepper-delegated `generate()` is **bit-identical** to the
  pre-stepper binary (same seed/text: 80640 samples, max abs diff 0).
- Sub-chunk SSE (`stream_frames_per_event=16`, margin 8): 6 deltas for 7.2 s
  audio, first at t+3.0 s warm (t+11 s cold incl. lazy load), then ~1.3 s
  apart. Concatenated bytes == offline byte count.
- Intermediate events are NOT bit-exact (max diff 563 int16 early, decaying
  to 0 at completion). Root cause: the codec decoder starts with a
  Transformer over the code sequence (`speech_decoder.cpp`: `transformer_norm`,
  `post_attention_layernorm`) — global attention means prefix decodes differ
  from full decodes everywhere, converging as the prefix grows. Final event
  (full prefix) is exact. Playback verified perfect by ear; diffs are
  inaudible perturbations, not artifacts.
- Default (`stream_subchunk` off) path byte-unchanged: single delta.
- `ctest`: 40/40 green.
- Codec frame rate: 1920 samples/frame (12.5 Hz @ 24 kHz).

**Embedded-spec caveat:** the published Breeze GGUF embeds its own v1
contract (`@gguf` wins over workspace `model_specs/`), so the three new
options are rejected as unknown until the GGUF spec is re-embedded. Testing
used `--model-spec-override <model_specs dir>`. Upstream PR must call this
out (re-embed specs at publish time, or document the override flag).

## Risks

- Conv-decoder prefix artifacts may need K larger than the real-time
  budget likes (latency floor = K frames + M frames). Fallback: shrink M,
  accept higher decode overhead.
- Depth-decoder per-frame state must be exactly resumable; verify
  bit-parity of stepped vs monolithic code sequences.
- Upstream may prefer this behind an explicit opt-in option rather than
  changing streaming granularity defaults — keep chunk-granular as default,
  sub-chunk opt-in via `stream_frames_per_event`.

## Work plan

1. Stepper refactor + monolithic wrapper (offline parity test).
2. Prefix-decode callback + margin emission in session; ear-tune K/M.
3. Options plumbing + spec entries + docs.
4. Full validation matrix above, then rebase onto `dev` and open follow-up PR.
