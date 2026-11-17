# CLAUDE.md — working memory for mcfrs-dispatch-transcribe

Dispatch radio audio in (from `mcfrs-scanner`), structured dispatch events
out (to `mcfrs-incidents`). Local speech-to-text plus matching to closed
lists. Purpose and data flow: `README.md`.

**Resume from `docs/RESUME.md` (status, next steps, gates).**

## Working agreement

Gates, TDD and commit format: `~/.claude/CLAUDE.md`. Cross-project conventions
(cite, don't restate; hollow checks; the Pi is exclusive to the main thread):
`~/code/CLAUDE.md`. Python uses `uv`.

Authorities for literal values:
- **This repo** owns the dispatch event format (not yet written).
- **Box facts** (host, RAM, ports in use): `mcfrs-scanner/CLAUDE.md`.
- **Talkgroups:** `mcfrs-scanner/config/mcfrs.tsv`.
- **Call types and unit nomenclature:** `mcfrs-incidents/api/mcfrs_api/parsers/`.

## What must never happen

- **Never commit audio or transcripts of real calls.** They contain addresses
  and patient detail. They live in `corpus/`, which is gitignored. Model
  weights go in `models/`, also gitignored.
- **Never use a cloud speech API.** Frank's constraint, 2026-11-17
  (`docs/adr/0001-speech-to-text-engine.md`).
- **Never touch `mcfrs-scanner`'s units, `sdr-mode` or the dongle from here.**
  This repo only reads the audio files the scanner writes. Hardware work on
  `apogee-gs` happens from that repo.
