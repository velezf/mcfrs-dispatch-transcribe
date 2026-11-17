# mcfrs-dispatch-transcribe

Turns Montgomery County (MD) Fire/Rescue dispatch radio calls into
structured dispatch events: units, call type, address. It sits between two
other repos:

```
mcfrs-scanner              mcfrs-dispatch-transcribe              mcfrs-incidents
(OP25 on apogee-gs)  →  per-call audio + metadata  →  speech-to-text  →  match to
                         (talkgroup, time)             (local engine)     closed lists
                                                                          →  dispatch events  →  source adapter → dashboard
```

- **Input:** one audio file per dispatch call, with talkgroup and time,
  recorded by `mcfrs-scanner` (its PLAN.md, phase 7).
- **Transcription:** a local speech-to-text engine. No cloud services.
  Which engine is decided by a bake-off on MCFRS audio
  (`docs/adr/0001-speech-to-text-engine.md`).
- **Matching:** the raw transcript is snapped to closed lists (unit roster,
  call types, county address points). The matching carries most of the
  accuracy; the transcript alone is not the product.
- **Output:** structured dispatch events, consumed by an `mcfrs-incidents`
  source adapter (its `docs/adr/0001-incident-source-without-everbridge.md`).
  This repo is the authority for the event format.

Scope is **dispatch**. Fireground ops traffic is too conversational and noisy
to parse; for ops, talkgroup activity is what the scanner already shows.

Not affiliated with Montgomery County or MCFRS. Encrypted traffic is never
decoded (`mcfrs-scanner` mutes it upstream).

## Status

Planning. Nothing runs yet. Live status and next steps: `docs/RESUME.md`.

## Privacy

Dispatch audio and transcripts contain real addresses and sometimes patient
detail. Recordings and reference transcripts live in the gitignored `corpus/`
and never enter the repo. Anything shown downstream goes through
`mcfrs-incidents`' server-side privacy transforms.
