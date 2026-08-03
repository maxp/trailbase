# Контракт: frontend build, migrations и deploy

Часть [Implementation Contract](../IMPLEMENTATION-CONTRACT.md). При противоречии с другими документами действует этот контракт.

## 19. Static assets и frontend build

- htmx, CSP-compatible Alpine, MapLibre bridge и styles поставляются локально.
- Frontend assets собираются `esbuild` через npm; committed `package-lock.json`,
  только `npm ci`.
- Node используется только в multi-stage build и отсутствует в runtime image.
- Content-hashed assets получают
  `Cache-Control: public, max-age=31536000, immutable`; HTML — `no-cache`.
- Assets отдаёт приложение из classpath. Caddy не требует shared static volume.

## 20. Миграции, CI, deploy и backup

- Migrations применяет one-shot `migrate` service до `web` и workers.
- Автоматического schema rollback нет. Down migrations тестируются, но запускаются
  вручную.
- Schema evolution следует expand/contract: additive migration, code rollout,
  отдельный backfill, удаление legacy в следующем release.
- PostgreSQL/PostGIS, Valkey, Caddy, MinIO и build/runtime images не используют
  `latest`. Production pin-ит version и digest.
- Test commands:
  - `bb test`: unit/contract без containers;
  - `bb test-integration`: real PostgreSQL/Valkey и local Telegram/Max HTTP stubs;
  - `bb test-all`: обязательный pre-merge gate;
  - `bb ci`: локальный эквивалент CI pipeline.
- Provider fixtures обезличены и включают positive, malformed и unknown event cases.
- GitHub Actions запускает migrations up/down, tests и production image build.
- Image публикуется в GHCR из protected `main` под immutable commit SHA.
- Production deploy запускается вручную через protected GitHub Environment с approval
  и явным SHA.
- Deploy использует SSH под ограниченным `trailbase-deploy`; production не содержит
  self-hosted GitHub runner и не собирает source.
- Хранятся current и previous successful image SHA. Application rollback переключает
  image назад без schema rollback.
- Post-deploy gate: service readiness, live/ready endpoints, migration version,
  worker heartbeat и public-page smoke. Провал автоматически возвращает previous
  application image и сохраняет diagnostics.
- Valkey не резервируется. PostgreSQL: daily full backup и continuous WAL archive.
  MinIO: versioning плюс off-host backup/replication.
- Raw, физически удалённый из primary MinIO, может сохраняться в encrypted off-host
  backup не более 30 дней после primary purge. Он недоступен приложению и не
  восстанавливается в active service.
- Initial targets: RPO до 15 минут, RTO до 4 часов.

## 21. Нормативные внешние ссылки

- [Creative Commons Attribution 4.0](https://creativecommons.org/licenses/by/4.0/)
- [Open Data Commons ODbL 1.0](https://opendatacommons.org/licenses/odbl/1-0/)
- [OpenStreetMap Foundation Tile Usage Policy](https://operations.osmfoundation.org/policies/tiles/)
