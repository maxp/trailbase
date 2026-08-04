---
label: wayfinder:map
title: TrailBase product concept — personas & use cases
---

## Destination

A new doc, `docs/overview/product-concept.md`: one paragraph on TrailBase's core
concept, its user personas (Contributor, Seeker, Moderator) with their confirmed
use cases, and a product-level summary of the classification/geo taxonomy
(seasons, movement modes, difficulty, Region/District/Location) that those use
cases rely on. Linked from `docs/README.md`'s overview row.

## Notes

- Domain: product framing and taxonomy for TrailBase, a public GPX track catalog
  with web + Telegram/Max bot interfaces (see `docs/overview/domain.md`,
  `docs/overview/system-map.md`).
- Consult `docs/adr/A06-classification.md` + `docs/contract/classification.md`,
  and `docs/adr/A04-poi-gazetteer.md` + `docs/contract/poi-map.md` before touching
  any classification/geo ticket — several tickets here amend those ADRs.
- Skills: `mattpocock-skills:grilling` and `mattpocock-skills:domain-modeling` for
  every ticket; use them to keep terminology consistent with the existing domain
  model rather than inventing parallel vocabulary.
- **Plan override:** this effort carries execution into the map (see "Plan, don't
  do"). The final ticket delivers the actual doc once the decision tickets below
  close — it isn't split into a separate build phase.
- **Standing rule (from AGENTS.md):** any ticket here that changes a decision
  already marked Accepted in an ADR must add a dated "Уточнение" entry to that ADR
  and update the matching Implementation Contract topic in the same session that
  resolves the ticket — not as separate follow-up work.
- Local-markdown tracker convention (no issue-tracker integration configured for
  wayfinder in this repo): each ticket is a file under `tickets/` with frontmatter
  `status` (open/closed), `type` (`wayfinder:<type>`), `assignee` (`unassigned` =
  unclaimed), and `blocked_by` (list of ticket filenames). Claim by setting
  `assignee`; resolve by adding an `## Answer` section, appending a resolution
  note, and setting `status: closed`.

## Decisions so far

(none yet)

## Not yet specified

- How the new taxonomy pieces interact in search/filter UI once tickets 02, 03 and
  04 land (e.g. filtering by simplified difficulty vs. per-activity standard, or by
  Region/District vs. individual POI) — depends on how those tickets resolve.
- Monetization/business model — not raised in the concept discussion so far;
  unclear whether it belongs in this doc at all.

## Out of scope

(none yet)
