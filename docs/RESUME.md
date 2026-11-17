# RESUME — mcfrs-dispatch-transcribe

## Status (2026-11-17)

Repo created; planning only. No code, no corpus. Public on GitHub
(`velezf/mcfrs-dispatch-transcribe`).

Decisions so far:
- `docs/adr/0001-speech-to-text-engine.md` (PROPOSED): local engines only;
  bake-off candidates Parakeet-TDT 0.11B, Whisper `small.en`/`base.en`,
  Qwen3-ASR-0.6B stock and the trunk-reporter P25 fine-tune; scoring method.
- `docs/adr/0002-phase2-vocoder-parameter-asr.md` (DEFERRED): no Phase II
  vocoder-parameter model for now; revisit triggers listed.

## Blocked on

- **`mcfrs-scanner` per-call audio tap** (its `docs/PLAN.md`, phase 7 step 2).

## Next steps

1. Frank: is MCFRS dispatch read by a person or by a synthesized voice?
   (It changes expected accuracy; ADR 0001.)
2. Not blocked, needs a go-ahead (installs software on `apogee-gs`): build
   antirez `qwen-asr` with OpenBLAS, download the P25 fine-tune, confirm it
   loads, time it on a test clip.
3. Record and hand-transcribe 20–30 dispatches into
   `corpus/`, then run the bake-off (ADR 0001, Decision).
4. Write the dispatch event format (this repo is its authority) once the
   bake-off shows which fields are reliable.

## Open branches

None.
