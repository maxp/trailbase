---
title: "Amend ADR A04 / POI contract: Region -> District hierarchy above Location"
status: open
type: wayfinder:grilling
assignee: unassigned
blocked_by: []
---

## Question

Add a two-level administrative geographic hierarchy, Регион (Region) → Район
(District), above the existing Location/POI entity. Confirmed: "Локация" at the
bottom of this hierarchy is the *same* entity ADR A04 already defines (the
`locations` table — Point/LineString/Polygon gazetteer entries with OSM
provenance); this ticket only adds the two grouping levels above it.

Still open:

- Are Region/District their own catalog entities — self-contained + OSM-provenance,
  mirroring how ADR A04 already treats POIs — or plain reference data seeded once
  from an external admin-boundary source and never re-synced?
- Who curates them: moderator-only creation/editing, like POI creation in ADR A04,
  or something else?
- Do tracks get assigned to a Region/District directly (a new field on the track or
  revision), or only transitively through the POIs a track already passes near
  (per ADR A04's auto-detect-along-track mechanism)?
- Does catalog browsing/search gain "browse by Region → District" as a primary
  navigation path (alongside the existing activity-type-first landing navigation
  from ADR A06), or is it a secondary filter only?

Resolve, then add a dated "Уточнение" entry to `docs/adr/A04-poi-gazetteer.md` and
update `docs/contract/poi-map.md`, in the same session.
