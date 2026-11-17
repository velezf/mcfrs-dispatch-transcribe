# ADR 0003 — Control-channel metadata as a second structured source

Status: PROPOSED 2026-11-17.

## Context

Dispatch voice transcription (ADR 0001) gives the call type and address,
with recognition errors. It does not reliably give *which units are working
which incident over time*. Two other routes are closed:
- Over-the-air dispatch data was checked and is absent
  (`mcfrs-scanner/docs/adr/0002-over-the-air-dispatch-data.md`).
- Member-only CAD relay products are ruled out on their terms
  (`mcfrs-incidents/docs/adr/0001-incident-source-without-everbridge.md`).

The P25 control channel is unencrypted signalling. It is already decoded in
full: 20,522 TSBKs in the 15-minute survey on 2026-11-17. It is
machine-readable, so it has no recognition errors. The fields that matter
here:

| Signal | What it says | Seen 2026-11-17 |
|---|---|---|
| Group voice grant (`0x00`) and grant update (`0x02`) | talkgroup *X* is active, keyed by radio *R*, at time *t* | OP25 logs `grp_v_ch_grant … ga: <tg> sa: <radio>`: 1,127 grants and 3,392 updates in the survey |
| Group affiliation response (`0x28`) | radio *R* joined talkgroup *X* | SDRTrunk decoded `GRP_AFFIL_RESP TO:<radio> GROUP ADDRESS:<tg>` from the recording. Whether OP25's single-receiver `trunking.py` logs these is unverified |
| Unit registration / deregistration (`0x2c` / `0x2f`) | radio *R* came onto or left the system | 645 / 435 in the survey |

MCFRS's talkgroup plan makes this useful (`mcfrs-scanner/docs/SYSTEM.md`,
Talkgroups). `7A2 Disp` (4005) is county-wide dispatch, and `Ops` and
`Inc NN` talkgroups are assigned to incidents. So "these radios moved onto
`Inc NN` right after a dispatch on 4005" is a units-to-incident signal.

## Proposal

Fuse two streams into incidents:

1. **Dispatch transcript** (ADR 0001): call type, address (matched to county
   address points), units named in the dispatch, time.
2. **Control-channel events** (this ADR): grants, affiliations and
   registrations, with talkgroup, radio ID and time.

Fusion rules (provisional; to be checked against real data):
- A dispatch on 4005 opens an incident at time *t*.
- The incident's working talkgroup is the `Ops`/`Inc` talkgroup named in
  the dispatch text, if the dispatch names one. (Assumed from common fire
  dispatch practice; to verify against MCFRS recordings.) Failing that, it
  is the first such talkgroup that radios affiliate to or key on within a
  few minutes of *t*.
- Units on the incident = radios affiliated to or keying on that talkgroup
  while it is active. An incident closes when its talkgroup goes quiet for
  a configurable period.
- Every inferred link carries its evidence (which grant or affiliation)
  and a confidence. The dashboard shows inferred assignments as inferred.

**Radio ID → unit name** is the weak link. There is no public alias list:
RadioReference removed it by policy, and IDs changed at the P25 cutover
(`mcfrs-scanner/docs/UI.md`). The box's
`/etc/mcfrs-scanner/radio-ids.tsv` holds 1 entry today. Transcription
closes the gap. When radio *R* keys up (`sa` in the grant) and the
transcript says "Engine 714 …", propose *R* → `E714`. Proposals are
accepted by hand at first; once a numbering pattern emerges, a single
`re:` line covers the fleet. This is the same `radio-ids.tsv` format the
listening page uses, so there is one file and one authority.

## Where the pieces live

- **`mcfrs-scanner`** emits control-channel events, alongside the planned
  per-call audio tap (its `docs/PLAN.md`, phase 7). The scanner owns OP25
  and the dongle.
- **This repo** fuses events with transcripts and owns the event and
  incident formats (`CLAUDE.md`, authorities).
- **`mcfrs-incidents`** consumes fused incidents through its source adapter.

## Constraints and limits

- **One dongle means control-channel gaps.** OP25 single-receiver leaves the
  control channel while it follows a voice call (`mcfrs-scanner` ADR 0001).
  Affiliations and grants sent during a followed call are missed, though
  link-control updates on the voice channel partly cover grants. A second
  dongle parked on the control channel removes the gap. That is the "second
  dongle" item in `mcfrs-scanner/docs/PLAN.md` ("Later"), and this ADR is a
  reason to promote it. The cost is about $40 plus an antenna split or a
  second antenna.
- **Encrypted talkgroups: metadata is ignored.** Grants for `[ENC]`
  talkgroups (`mcfrs-scanner/config/mcfrs.tsv`: 7D/7E secure ops, hospital
  talkgroups) are visible in the clear on the control channel. Reading them
  is not decryption. But the county encrypts those talkgroups for a reason
  (patient information, secure operations), so their events are dropped at
  ingest and never stored or shown.
- **Personal use only.** Receiving unencrypted signalling is generally
  lawful (18 U.S.C. §2511(2)(g)(ii)); divulging it is restricted (47 U.S.C.
  §605). Not legal advice. Radio IDs and unit movements are not published.
- **LRRP location is out of scope.** The standard data channel carries
  unencrypted radio GPS reports (`mcfrs-scanner` ADR 0002, results (e)).
  Using responder positions would need its own ADR. It would also conflict
  with `mcfrs-incidents`' rule against claiming apparatus position.

## Alternatives

- **Voice transcription only:** misses unit movements and is weakest on
  exactly the unit designators this source gives exactly.
- **Control channel only:** gives who and when, but not what or where. No
  call type, no address.
- **LRRP positions:** rejected above (privacy, out of scope).

## Next steps (each a gate Frank clears)

1. **Verify on the box** that OP25's `trunking.py` logs `0x28` affiliation
   responses. If not, the scanner-side tap needs a small log addition (log
   lines only, as with the 2026-11-17 patches).
2. **Offline replay:** run the fusion rules over a control-channel log plus a
   handful of hand-transcribed dispatches, and check the inferred
   unit-to-incident links against what the audio says.
3. **Decide on the second dongle** from step 2's gap measurements.
4. **Write the event format** (this repo is its authority), together with
   ADR 0001's dispatch event format.

## Trigger to revisit

Step 2 shows the inference is unreliable, or MCFRS changes how it assigns
incident talkgroups.
