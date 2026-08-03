# TrailBase — Implementation Contract

Статус: принятые решения.

Это навигационный вход в нормативный контракт TrailBase. Контракт задаёт проверяемые
инварианты реализации и имеет приоритет над roadmap, overview и ADR. Изменение
принятого решения требует обновить соответствующий тематический контракт и ADR.

## Контракт по областям

- [Runtime и platform](contract/runtime.md)
- [Identities и account creation](contract/auth/identity.md)
- [Browser sessions, linking и delivery](contract/auth/browser-delivery.md)
  - [Browser session и identity linking](contract/auth/browser-and-linking.md)
  - [Browser auth flow и delivery-health defaults](contract/auth/browser-flow.md)
- [Valkey sessions и account lifecycle](contract/auth/sessions-lifecycle.md)
- [Permissions и HTTP security](contract/security-permissions.md)
- [Webhooks, bot workers и notifications](contract/bots.md)
  - [Notification delivery и provider health](contract/bots/delivery-and-provider-health.md)
- [Health, logs и metrics](contract/observability.md)
- [GPX intake, geometry и object storage](contract/tracks/ingest-storage.md)
- [Tracks, revisions и public exports](contract/tracks/revisions-exports.md)
  - [Track integrity и revisions](contract/tracks/integrity-revisions.md)
  - [Moderation и removal appeals](contract/moderation-appeals.md)
  - [Public GPX exports и licenses](contract/tracks/public-exports.md)
- [Classification и tags](contract/classification.md)
- [POI и map delivery](contract/poi-map.md)
- [Search](contract/search.md)
- [Frontend build, migrations и deploy](contract/platform.md)
  - [Monitoring и deployment control](contract/platform/deployment-control.md)

## Связанные документы

- [Документация: как выбирать документ](README.md)
- [System map](overview/system-map.md)
- [Roadmap](ROADMAP.md)
- [Architecture decisions](adr/)
