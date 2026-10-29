# ADR 0019 — LNS names in members-only lists, resolved to addresses at publish

Status: proposed (desktop view). Recorded 2026-10-28.

## Context
A members-only form lists respondent addresses (`whitelist.type = "addresses"`). The
creator has to collect a `0x` address from every person first, out of band. LNS names
(looked up through the optional `names` Basecamp module) give people a
name others can look up, with text entries the holder sets.

## Decision
- A person links once: their name's `whisperbox.address` text entry holds their
  WhisperBox address. Only the name's holder can set it, so the entry is the proof. The
  Identity screen shows the `names set-text` command with this device's address.
- In the builder's allow-list an entry starting with `0x` is an address, as before; any
  other entry is an LNS name. It is cleaned up exactly as the registry does it (LNS spec
  section 2: trim, lowercase, drop `lns:`, then `@`, then a trailing `.lez`), so `@Alice`
  and `lns:alice` mean `alice`. An entry that isn't a valid name after that stops the
  publish before any lookup.
- The creator's view resolves the names through `names.resolve` when publishing, saving an
  edit or scheduling, and puts only the addresses into the form. A name that is not
  registered, has no valid linked address, or can't be looked up (no `names` module) stops
  the publish with a message naming it; it is never dropped.
- Names are looked up one at a time, each with its own deadline, with progress on the
  Publish button: the `names` module answers one call at a time (about 0.4 s each), so
  parallel lookups of a long list all waited on the last one and timed out. The view
  also looks names up while the list is typed and keeps the addresses it finds while the
  builder is open, so most are done before Publish and problems show under the list.
  Failed lookups aren't kept; Publish asks again.
- `names` is an optional dependency of the view. The core, the wire format, the hub, the
  Android app and the SDK are unchanged.

## Consequences
- Every client and co-owner checks answers against plain addresses, so the accepted set is
  still deterministic (ADR 0003). A name whose entry changes later doesn't change a
  published form; the creator re-saves it to pick up the new address. A scheduled form uses
  the addresses resolved when it was scheduled.
- Names are kept only in the local draft's builder state. Editing a published form shows
  its addresses, not the names.
- Whitelists are public. Anyone who knows or guesses a name can look up its address and
  see whether that person is on a form's list. LNS names are private unless known or
  guessed; this is the same limit.
- Not combinable with anonymous forms, like any members-only list (ADR 0011).
- The lookup is not trustless. `resolve` believes the one sequencer the creator's `names`
  module asks (LEZ reads carry no proof). The exposure is that endpoint, on the creator's
  machine, at the moment of publishing; the published form holds plain addresses.
- The link is one-sided. The LNS spec (section 7) links a name to a Logos account only
  when both sides agree, because a holder can write any address into their name. Here one
  side is enough: the holder chooses which address answers for them, which is all an
  allow-list needs.
