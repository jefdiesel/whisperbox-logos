# ADR 0019 — Every answer is signed; deadlines use the network's timestamp

Status: accepted, implemented on `feat/signed-answers` (target 0.5.0). Recorded 2026-10-22. Supersedes the HLC-based enforcement in
ADR 0008 and ADR 0010, and the unsigned answers of ADR 0003 for forms open to anyone.

## Context
The 2026-10-22 review (WB-2) showed that on a form open to anyone an answer is unsigned and
carries an HLC the sender chooses. A member can send an answer in another respondent's name (the
earliest HLC per respondent wins, so the real answer is dropped), stamp an answer before a close,
an expiry or a receipt, and fill `maxResponses` with old stamps. Only "listed addresses" and
editable forms sign inside the sealed answer.

Logos Delivery already has a time a sender can't choose freely: the message timestamp. With RLN
on, every relay rejects a message whose timestamp is more than ~20 s from its own clock, and the
timestamp is bound into the RLN proof (logos-delivery `rln/rln_evm/rln_evm.nim`,
`MaxClockGapSeconds = 20`). Store nodes keep the message with that timestamp, so a creator who
comes back online later still sees when each answer was really published. RLN is off on our
nodes today and will be on: there's no way around it.

## Decision
1. **Every answer is signed** inside the sealed payload, with the inner signature that "listed
   addresses" and editable forms already use. An anonymous form signs with the respondent's
   per-form identity. The event itself stays unsigned, so nothing on the wire names the
   respondent. A creator view counts an answer only if its signature verifies and the signer is
   the `respondent` it names; unsigned answers (from clients before 0.5.0) are shown as
   "unverified" and never count towards a cap, a duplicate check or an edit.
2. **Deadlines use the network timestamp.** For each answer the creator keeps the earliest network
   timestamp it has seen for that event (live or from the store; a peer re-serving it during
   catch-up publishes a newer copy, so the earliest one is the original). Close, expiry, the cap
   and the receipt lock compare that time, not the answer's own HLC. The respondent's claimed time
   (inside the signed payload) is for display; if it is more than a few minutes from the network
   time, the answer is flagged.
3. **Without a network time** (a bearer that has none, such as the BLE mesh, or a Loam too old to
   pass it) the answer's own HLC is used, as in 0.4, and the answer is marked "time not
   verified". (First proposed: the creator's first-seen time. Rejected while implementing: a
   creator who was offline sees honest answers first during catch-up, after the close, and would
   drop them all.) Until RLN is on, the network time is only as honest as the sender's node; the
   5-minute future bound (0.4.3) still applies.
4. **Loam passes the time.** loam_core already hands apps the message timestamp (ns); it must also
   deliver an earlier-stamped duplicate (the store copy arriving after a re-served one), and
   loam-transport must pass `{ts, source}` to `onMessage` on the phone, including through the
   shared Loam app. Requested from the Loam owner 2026-10-22.

## Consequences
- Impersonating a respondent and replacing their answer stop working on every form.
- One answer per identity is enforced; throwaway identities are still possible on open forms
  (only a members list or token gating stops those).
- Close, expiry, cap and receipt lock become as exact as the network's clock once RLN is on.
- A protocol change: 0.4.x clients' answers are "unverified" to a 0.5.0 creator. Creators should
  update first; the view says why an answer is unverified.
- Answers relayed over BLE, or by a Loam without timestamps, can't be time-checked.
- The C++ core and the JS client change together, with new conformance fixtures and the hostile
  cases (forged signer, backdated HLC, late answer with an early claim) in the parity and e2e tests.

## Implementation (2026-10-22)
- A network time is an upper bound on when the answer existed: any copy's timestamp proves "sent
  by then", none proves "sent after". The earliest copy is the best one; a creator that never
  sees the original or its store copy, only a re-served one, judges an honest answer late.
- Engine: `creatorView(..., netTime)` (JS opts, C++ last parameter). Fields `timeVerified`,
  `receivedAt`, `timeSkew`. Golden vector `netView` in `golden-lifecycle.json`.
- Desktop: loam_core `received(topic, sender, payload, ts)` (ns). loam_core 0.8.0 passes 0 when
  there's none (the channel event's time is the local receive time, so it counts as none) and
  delivers an earlier-stamped duplicate again; older loam_core passes the WakuMessage time on the
  relay path WhisperBox uses. Kept in `nettime.json`; diagnostic `netTs`.
- Phone: loam-transport >= 86b6bb3 passes `onReceive(topic, candidates, {ts, tsMs, source, bearer})`
  (and through the shared Loam app >= 0.0.57); the client keeps `wb-nettime`. An older transport
  passes nothing: every answer is "time not verified".
- Views show the network time, "time not verified", or the sender's clock when it is off.
