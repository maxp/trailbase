---
title: Reputation accrual formula
status: closed
type: wayfinder:grilling
assignee: maxp
blocked_by: []
---

## Question

Reputation accrues from three action types, per the concept dialogue:

1. Using the service over a time interval (exact trigger unclear — a login? a
   session of some minimum length? a recurring daily/weekly tick?).
2. Leaving positive feedback (no feedback/review/rating mechanism exists anywhere
   in the current domain model — see the map's Notes on the coupling to the
   sibling product-concept map. Define, at minimum, what "positive feedback"
   means for Reputation's purposes: feedback on what entity — a track? a
   Contributor? — and how "positive" is determined.)
3. Uploading a track.

Resolve:

- Exact accrual amount (or formula) per action type — flat per action, scaled by
  something (e.g. track quality/integrity status), or otherwise.
- Caps or diminishing returns (does repeated action N keep paying out, or taper)?
- Anti-gaming resistance: how is "using the service over an interval" prevented
  from being gamed by an idle open tab or scripted polling; how is "positive
  feedback" prevented from being gamed by reciprocal/fake-account feedback loops?
- Does accrual live on the Account (matching how Account already owns
  "preferences" per `docs/overview/domain.md`) rather than per-Identity — confirm
  this assumption.

## Answer

Reputation lives on the Account (per the existing "Account owns preferences"
pattern — not re-litigated, taken as a safe default).

Three accrual triggers, each a flat amount (exact magnitudes are tunable
constants, not fixed by this decision):

1. **Daily-use tick** — once per calendar day per Account, triggered by any
   authenticated request that day (no minimum session length, no bonus for more
   activity the same day). Anti-gaming: an idle open tab earns nothing, since it
   requires an actual authenticated request, not just a connection.
2. **Track publication** — fires when a revision that includes a description
   passes moderation and becomes public (not on raw upload). No additional cap or
   diminishing returns: moderator approval is already the rate limiter, so
   publishing many good tracks just pays out every time.
3. **Confirmed-useful comment** — introduces a new domain concept, **comments on
   a published track**, purely for this feedback purpose (mechanics of the
   comment feature itself — moderation, storage, display — are out of this
   ticket's scope; see the newly-created ticket "Comment feature: storage,
   moderation & display mechanics"). A comment becomes "confirmed useful" once
   either the track's own author confirms it, or **≥3 distinct Accounts**
   (excluding the author) confirm it *and* their summed Reputation crosses a
   threshold T. On crossing, the commenter gets a flat one-time reward.
   Confirmers also get a small reward each, to make confirming worthwhile:
   - Decays per repeat (confirmer, commenter) pair (e.g. 1.0×, 0.5×, 0.25×, …)
     rather than cutting off after the first — kills two-account loop-farming
     quickly while still letting a genuine repeat mentor relationship pay a
     little.
   - Capped to each confirmer's first N reward-eligible confirmations per day,
     independent of who they're confirming — blocks bulk/indiscriminate
     confirming as a farming strategy.
   - The ≥3-distinct-and-summed-reputation quorum itself is the main
     "genuinely useful to the community" filter: it can't be satisfied by one
     account's opinion regardless of that account's own Reputation.
