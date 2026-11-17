# ADR 0001 — Dispatch speech-to-text: local engines, chosen by a bake-off

Status: PROPOSED 2026-11-17. Moved here from `mcfrs-scanner` (drafted there as
ADR 0003, never committed). The candidates and the test are fixed here; the
engine is chosen only after the bake-off runs on MCFRS audio.

## Context

With no Everbridge feed (`mcfrs-incidents` ADR 0001) and probably no
over-the-air dispatch data (`mcfrs-scanner` ADR 0002), the realistic live source for a
personal dashboard is the dispatch voice `mcfrs-scanner` already decodes,
turned into structured incidents.

Constraints (Frank, 2026-11-17):

- **Local only.** No cloud speech APIs. The audio is county dispatch traffic,
  and a third-party service would be paid per minute and would see all of it.
- **Runs on `apogee-gs`** (Pi 5, 8 GB, `mcfrs-scanner/CLAUDE.md` box facts), next to
  OP25 and the rocket ground station. The Mac is the fallback compute.

What makes it tractable:

- **Dispatch uses a rigid spoken template** (alert tone, box area, units,
  call type, address, cross street, talkgroup). The transcript is not the
  product. It gets snapped to closed lists: the unit roster, the call-type
  table in `mcfrs-incidents` (`api/mcfrs_api/parsers/`), and the county
  address points for geocoding. This matching probably decides accuracy more
  than the choice of engine.
- **Low duty cycle.** Calls are short and sparse, so each one is transcribed
  after it ends (batch), not streamed. Slower-than-real-time engines are
  acceptable.
- **Personal use tolerates errors** if each parsed incident keeps its audio
  clip for replay.

What works against it:

- P25 Phase II audio is AMBE+2 at 2.45 kb/s, which general models handle
  poorly. Sirens and alert tones overlap the speech, and unit designators and
  street names are where generic models fail.
- **Fireground ops traffic is out of scope.** It is conversational, noisy and
  full of cross-talk. For ops the target is metadata (active talkgroup, units
  keying), which the listening page already has.
- One dongle, shared with the I-Gate: dispatch is heard only in scanner mode.
  The "second dongle" item in `mcfrs-scanner/docs/PLAN.md` ("Later") removes that.
- Is MCFRS dispatch read by a human or by a synthesized voice? Frank to
  confirm. A synthesized voice would raise accuracy sharply.

## Options

Researched 2026-11-17. Published numbers come from clean speech on other
hardware; none are radio audio. They rank candidates for testing only.

| Engine | Runtime on Pi 5 | Evidence | Verdict |
|---|---|---|---|
| **NVIDIA Parakeet-TDT 0.11B** | sherpa-onnx | ~4.2 % WER, RTF 0.12, 1.2 GB RAM on a Pi 5, clean speech, one write-up [1] | **test** (speed/accuracy leader) |
| **Whisper `small.en` / `base.en`** | whisper.cpp | Pi 5: `base` ≈ real time, `small` below real time [2][3]; base-int8 via sherpa-onnx ~11 % WER, RTF 0.34 [1]. An initial prompt can bias vocabulary | **test** (baseline). [trunk-transcribe][4] is a working trunk-recorder → Whisper pipeline to borrow from |
| **Qwen3-ASR-0.6B (stock)** | antirez `qwen-asr` (C, MIT, NEON, ~2.8 GB RSS) [5], or community GGUF/ONNX [6][7] | Matches Whisper large-v3 on English benchmarks at a fraction of the size. Strong on the vendor's own noise sets (17.9 vs 63.2 WER, Mandarin) [8]. Prompt biasing is "very soft" [5]. No Pi numbers; ~8× real time on an M3 Max CPU [5], so Pi 5 is estimated near real time (unmeasured) | **test** |
| **`trunk-reporter/qwen3-asr-p25-0.6B`** | same as stock, if the fine-tune loads (unconfirmed) | Apache-2.0. Decoder-only fine-tune on ~24.8 k calls from unnamed county P25 systems, labelled by other models (pseudo-labels, not human-verified). No WER published, "informal evaluation" only. 111 downloads/month; code repo "coming soon" [9] | **test** against stock Qwen: only worth keeping if it beats it on MCFRS |
| trunk-reporter IMBE-ASR | 70 KB C binary, 15× RT on Pi 5 [10] | Reads Phase I IMBE vocoder parameters. MCFRS is Phase II AMBE+2, so it does not apply | reject. A Phase II equivalent: ADR 0002 |
| Moonshine | sherpa-onnx / own runtime | Optimized for streaming latency [11] | skip: we transcribe finished calls, so latency buys nothing |
| Vosk (Kaldi) | native | Older and less accurate. Its grammar restriction is the draw, but reports are mixed [12] | skip: post-matching to closed lists does the same job on a better model |
| Cloud APIs | — | often most accurate | reject: local-only constraint |

Research on police radio supports domain fine-tuning. Off-the-shelf large
models do poorly, and fine-tuned models approach human range [13]. A
commercial emergency-radio product made the same claim for fine-tuned
Whisper [14].

## Decision

Run a bake-off on MCFRS audio, on the Pi, before choosing:

1. **Corpus:** 20–30 dispatch calls (talkgroup per `mcfrs-scanner/config/mcfrs.tsv`)
   recorded from OP25's decoded audio, with a hand-written reference
   transcript for each. Recordings and transcripts contain real addresses:
   they live in the gitignored `corpus/` and never enter any repo.
2. **Candidates:** Parakeet-TDT 0.11B, Whisper `small.en` (with a vocabulary
   prompt) and `base.en`, Qwen3-ASR-0.6B stock, and the P25 fine-tune.
3. **Score what the dashboard needs, after matching to the closed lists.**
   For each call, check the units, call type and address. The address passes
   when it geocodes to the right address point. Raw WER is reported
   alongside but does not decide. Also record wall time per call and peak
   RSS on the Pi with OP25 running.
4. **Pick** the best field accuracy that keeps up with dispatch volume. If
   none is good enough, the next step is fine-tuning the winner on a few
   hundred MCFRS calls, not a different engine.

Integration shape (provisional): `mcfrs-scanner` saves one audio file per
dispatch call with its metadata (talkgroup, time); this repo transcribes and
matches it and emits structured dispatch events; `mcfrs-incidents` consumes
them through a new source adapter. The event format's authority is this repo
(the producer); the adapter cites it.

## Consequences

- Needs a working 800 MHz antenna first (`mcfrs-scanner` ADR 0002 evidence).
- Prep that needs no antenna: build `qwen-asr` with OpenBLAS on the box,
  download the P25 fine-tune (~2 GB), and check that it loads and how fast
  it runs on a test clip. This installs software on a box with live
  services, so it needs Frank's go-ahead.
- Transcription saturates the Pi's four cores in bursts. Watch for OP25
  audio underruns during the bake-off.

## Trigger to revisit

The bake-off results are in, or a local engine with published Pi 5 numbers
on P25 audio appears.

## Sources

1. https://hashnode-hub.vercel.app/posts/68262037cbfffe5887227bb0
2. https://www.promptquorum.com/power-local-llm/local-whisper-stt-comparison-2026
3. https://github.com/ggml-org/whisper.cpp/discussions/166
4. https://github.com/CrimeIsDown/trunk-transcribe
5. https://github.com/antirez/qwen-asr
6. https://huggingface.co/handy-computer/Qwen3-ASR-0.6B-gguf
7. https://huggingface-proxy.pages.dev/Daumee/Qwen3-ASR-0.6B-ONNX-CPU
8. https://huggingface.co/Qwen/Qwen3-ASR-0.6B
9. https://huggingface.co/trunk-reporter/qwen3-asr-p25-0.6B
10. https://trunk-reporter-readme.static.hf.space/README.md
11. https://moonshine-voice.readthedocs.io/en/latest/moonshine-vs-whisper/
12. https://github.com/rhasspy/wyoming-vosk
13. https://www.arxiv.org/pdf/2409.10858
14. https://www.strv.com/our-work/citizen
