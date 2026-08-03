# TrailBase — System map

TrailBase is a public GPX catalog with a web interface and Telegram/Max chat
interfaces. Web and bots use the same domain services; PostgreSQL/PostGIS is the
source of durable application state.

## Runtime boundary

```text
[Telegram/Max] ─ identity, upload, search ─┐
                                             ▼
[Web: htmx + Alpine + MapLibre] ─────▶ [Clojure backend]
                                             │
                      ┌──────────────────────┼──────────────────────┐
                      ▼                      ▼                      ▼
               PostgreSQL/PostGIS      S3-compatible store        Valkey
                accounts, tracks,       private raw/export       sessions, streams
                 revisions, search
```

## Fixed technology choices

- PostgreSQL + PostGIS holds relational, spatial and search state.
- S3-compatible storage holds exact private raw uploads and derived objects.
- Valkey holds ephemeral sessions, tokens and streams; it is not the durable source
  for ownership or track state.
- The web is server-rendered with htmx and Alpine; MapLibre is an isolated map
  component.
- Telegram and Max are the only identity providers in the MVP.

For exact interfaces and invariants, read the [Implementation Contract](../IMPLEMENTATION-CONTRACT.md).
