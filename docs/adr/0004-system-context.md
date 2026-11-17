# ADR 0004 — System context: this repo's role

Status: PROPOSED 2026-11-17. The full system map is canonical in
`mcfrs-incidents/docs/adr/0002-system-architecture.md`; this ADR states only
this repo's place in it.

## Role

The interpreter: it turns radio into structured dispatch events. It runs on
`apogee-gs` beside the scanner but never touches the dongle.

```
 mcfrs-scanner ─► per-call audio + metadata ┐
               └► control-channel events ───┤
                                            ▼
 mcfrs-dispatch-transcribe: local speech-to-text (ADR 0001)
   ─► match to closed lists (units, call types, county address points)
   ─► fuse with control-channel events (ADR 0003)
   ─► dispatch events ─► mcfrs-incidents (source adapter)
```

## Owns

- The **dispatch event format**: its output, read by `mcfrs-incidents`'
  source adapter.
- The speech-to-text engine choice and bake-off (ADR 0001), and the fusion
  rules (ADR 0003).

Clarification of ADR 0003: there, "owns the event and incident formats"
means *its output*. The control-channel event format belongs to
`mcfrs-scanner`, which produces it. The incident model belongs to
`mcfrs-incidents` (canonical map, "Roles and owned contracts").

## Reads, but does not own

- Talkgroups: `mcfrs-scanner/config/mcfrs.tsv`.
- Call types and unit nomenclature: `mcfrs-incidents/api/mcfrs_api/parsers/`.
- Station rosters: `mcfrs-incidents/api/mcfrs_api/data/stations.json`.

## Boundaries

- No cloud speech services (ADR 0001).
- Never commits real call audio or transcripts (`corpus/` is gitignored).
- Does not store or serve incidents. That is `mcfrs-incidents`.
- Does not touch the scanner's units or `sdr-mode` (`CLAUDE.md`).

## Trigger to revisit

The event format is written (it gets its own ADR here), or a source other
than the scanner starts feeding this repo.
