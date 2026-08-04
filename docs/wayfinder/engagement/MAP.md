---
label: wayfinder:map
title: TrailBase user engagement & motivation system
---

## Destination

A new `docs/adr/A09-user-engagement.md` (decision + rationale + alternatives,
following the existing ADR pattern) plus a new `docs/contract/engagement.md`
(normative rules), linked from `IMPLEMENTATION-CONTRACT.md`'s topic list. Covers
the motivation system as a whole; documents its first two mechanics — **Reputation**
and **Stamina** — with room for further mechanics (badges, streaks, etc.) to be
added later as their own subsections without renaming anything.

## Notes

- Domain: gamification/motivation mechanics for TrailBase. Reputation = a
  per-Account trust/standing score that accrues from activity and decays
  logarithmically over time; visible in the catalog, prioritizes the moderation
  queue. Stamina = a per-Account spendable resource, consumed by track uploads,
  regenerating over time at a rate tied to Reputation, and top-uppable with real
  money.
- Consult `docs/overview/domain.md` (Account/Identity model — these mechanics are
  assumed Account-level, matching how Account already owns "preferences"),
  `docs/adr/A01-bot-first-auth.md` (identity model, relevant to any payment flow
  through Telegram/Max), and `docs/adr/A06-classification.md` (existing FIFO
  moderation-queue mechanics, relevant to Reputation's moderation-priority effect).
- **Coupled to a sibling map:** [TrailBase product concept — personas & use
  cases](../product-concept/MAP.md) has an open ticket ("Confirm remaining persona
  use cases") asking whether social/feedback features exist at all. Reputation's
  "positive feedback" accrual trigger depends on that — there is currently no
  review/rating/feedback mechanism anywhere in the domain model. Don't wait on that
  ticket; the Reputation-accrual ticket here should define the minimal feedback
  concept it needs, but flag the coupling so the two don't diverge.
- Skills: `mattpocock-skills:grilling` and `mattpocock-skills:domain-modeling` for
  every HITL ticket; `mattpocock-skills:research` (background subagent) for the
  research ticket.
- **Plan override:** like the sibling map, this effort carries execution into the
  map — the final ticket writes the ADR + contract doc once the decision tickets
  close.
- **Standing rule (from AGENTS.md):** if resolving any ticket here turns out to
  require changing an already-Accepted ADR (e.g. the auth/session model, if a
  payment flow needs new identity-linked state), update that ADR/contract in the
  same session, not as separate follow-up work.
- Local-markdown tracker convention (same as the sibling map): each ticket is a
  file under `tickets/` with frontmatter `status`, `type` (`wayfinder:<type>`),
  `assignee` (`unassigned` = unclaimed), and `blocked_by`.

## Decisions so far

- [Research: payment mechanisms for topping up Stamina](tickets/05-research-stamina-payment-mechanism.md) —
  Telegram's classic Bot Payments API with YooKassa as provider is the
  strongest RUB-settling, fully in-chat rail; Max has no bot payments API at
  all (needs a standalone-gateway fallback); NPD self-employed registration is
  the lightest compliance fit up to 2.4M RUB/year.
- [Reputation accrual formula](tickets/01-reputation-accrual.md) — flat
  per-trigger accrual from a daily-use tick, track publication, and a new
  confirmed-useful-comment mechanic (≥3 distinct confirmers + summed-Reputation
  threshold, with decaying per-pair and daily-capped rewards for confirmers).
  Surfaced a new sub-feature (comments) that needs its own ticket.
- [Reputation decay formula](tickets/02-reputation-decay.md) — inactivity-based
  linear weekly decay (`R_start − w · t_weeks`), pausing during active periods,
  down to a floor that grows logarithmically with lifetime published-track count
  rather than a flat 0.

## Not yet specified

- Whether Reputation feeds into anything beyond moderation priority and catalog
  display (e.g. upload rate limits, search ranking) — not raised yet, too
  speculative to ticket.
- Any mechanic beyond Reputation and Stamina (badges, streaks, leaderboards,
  referral incentives) — explicitly deferred; the destination is scoped to these
  two for now (per the answer that confirmed Reputation/Stamina as the current
  full scope).

## Out of scope

(none yet)
