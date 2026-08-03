# M04 — Catalog Render

## Goal

Render the public catalog and its map island from published tracks.

## Acceptance

- Public pages expose only publishable track data.
- The map receives geometry through the defined adaptive delivery path.
- Browser state, caching and error behavior follow the contract.
- Catalog and map use the same information architecture and domain behavior in the
  browser, Telegram Mini App and MAX Mini App, with provider-specific theme, viewport,
  navigation and link handling verified in each messenger.

## Contract

- [Tracks, revisions и public exports](../contract/tracks/revisions-exports.md)
- [POI и map delivery](../contract/poi-map.md)
- [Telegram и MAX Mini Apps](../contract/miniapps.md)
