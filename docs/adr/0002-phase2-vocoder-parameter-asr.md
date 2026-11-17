# ADR 0002 — Speech recognition on Phase II vocoder parameters: deferred

Status: DEFERRED 2026-11-17. Moved here from `mcfrs-scanner` (drafted there as
ADR 0004, never committed).

## Context

trunk-reporter's IMBE-ASR models transcribe P25 Phase I by reading the IMBE
vocoder parameters directly instead of decoded audio. Their published
results: 48.6 M parameters at 15× real time on a Pi 5, but 18.9 % WER
(greedy) on their synthetic LibriSpeech-IMBE set; 290 M parameters at 1.9 %
WER with beam search, also on that synthetic set and not on real radio
(`0001-speech-to-text-engine.md`, source 10). Their code is not yet
public.

MCFRS is Phase II, with a different vocoder (AMBE+2 half rate), so those
models do not apply. Frank asked whether we could build the Phase II
equivalent.

## Feasibility

Possible with tools already on the box:

- OP25 (installed on `apogee-gs`, see `mcfrs-scanner`) decodes each Phase II voice frame into MBE parameters (`lib/ambe.c`,
  called from `lib/p25p2_tdma.cc`) before synthesizing audio. Exporting
  them per 20 ms frame is a small OP25 patch.
- OP25 has an AMBE+2 encoder with the Phase II interleaver
  (`lib/ambe_encoder.cc`, `p25p2_vf`). That is how synthetic training data
  would be made.

Approach, mirroring trunk-reporter:

1. Encode a public speech corpus (LibriSpeech, ~1,000 h) to Phase II frames.
2. Train an ASR model on the frames.
3. Fine-tune it on real MCFRS frames, pseudo-labelled by the ADR 0001 winner
   and spot-checked by hand.

## Decision

Do not build it now.

- **No information gain.** Decoded audio is synthesized from exactly these
  parameters. Reading them directly avoids synthesis artifacts but cannot
  recover what the 2.45 kb/s vocoder discarded.
- **It gives up pretraining.** Audio models (Whisper, Parakeet, Qwen) were
  trained on tens of thousands of hours or more. A parameter-domain model
  starts from scratch on ~1,000 h of synthetic audio.
- **Its advantage is speed, which is not our bottleneck.** Dispatch is
  sparse; accuracy on units and street names is the problem.
- **Encoder mismatch.** OP25's encoder is an independent implementation,
  not DVSI's chip in real radios. Training on its output adds a domain gap
  that is hard to measure.
- **It is a research project** (pipeline, GPU time, labelling,
  evaluation), weeks to months of work.

The same domain-adaptation benefit comes more cheaply from fine-tuning an
audio model on MCFRS recordings (ADR 0001, decision step 4).

## Trigger to revisit

- The ADR 0001 bake-off shows audio models failing *specifically* on vocoder
  artifacts (errors that persist after fine-tuning on MCFRS audio), or
- trunk-reporter publishes its code or adds Phase II. Contributing there
  beats starting alone, or
- Frank wants it as a project for its own sake.
