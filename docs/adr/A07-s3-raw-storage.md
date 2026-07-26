# ADR A07 — Хранение raw GPX и фото в S3-совместимом хранилище

**Status**: Accepted
**Date**: 2026-07-25

## Контекст

PostGIS хранит распарсенную geometry треков и метаданные. Но нужен durable источник для (а) re-parse (если improved-attribute extractor появится позже), (б) скачивания пользователем, (в) фото трека. PostGIS-geometry — это processed projection, а не источник истины. БД не подходит для blob-хранилища — твёрдый паттерн `geometry in DB, raw in object storage`.

## Решение

1. **S3-совместимое хранилище** для raw GPX и фото (MinIO / Garage / Cloudflare R2 — self-hosted совместимо).
2. **Доступ из Clojure-бэкенда**: **aws-simple-sign** (SigV4 подпись, zero-dep) + **hato** (transport с прокси) — см. A08. Subiesto одной HTTP-стеке для bot API, S3, и external fetches.
3. **Паттерн**:
   - Upload: при загрузке трека через бота/веб — raw GPX записывается в S3 (`tracks/{track_id}.gpx`), затем backend парсит geometry в PostGIS, хранит `s3_uri` в `track`-row.
   - Download: backend генерит presigned URL (aws-simple-sign), клиент скачивает на прямую из S3.
   - Фото: `{track_id}/photos/{n}.jpg` (public-catalogue или thẩm privated), отдаются через presigned URL или CDN.
4. **Re-parse pipeline**: backend читает из S3 (через hato), парсит `data.xml`, обновляет PostGIS-geometry и attributes. Независим от web-trafika.

## Альтернативы рассмотренные

- **Хранить raw GPX в BД (bytea).** Отвергнуто: BDb не для blob, backup/replication cost, query/scan perf деградирует.
- **AWS SDK v2 Java interop.** Отвергнуто: ~15-30 MB dep, Netty/Reactor, перекладка через abstraction-layer; избыточен для 3-4 операций (upload, GET, presigned, delete).
- **Cognitect aws-api.** Отвергнуто: легче SDK, но надстройка; для узкого набора операций тяжёлый service-definitions artifact; issue с медленным client creation и erased-types interop.
- **Amazonica / clj-aws-s3 (weavejester).** Отвергнуто: поверх AWS SDK v1 (deprecated, EOL).
- **Filesystem (volume mount).** Приемлемо для very-MVP, но не distributed-ready; масштаб 2 подразумевает multi-node-ready. S3-совместимость — upgrade path.

## Последствия

- Положительные: raw не дублируется в БД и filesystem; durable; S3-compat = multi-region/CDN-ready; одна HTTP-стек (hato) для всего; aws-simple-sign = десятки KB против десятков MB.
- Отрицательные: eventual consistency между S3 upload и DB write (нужен upload-then-insert ordering, cleanup on failure); presigned-URL expiry management; MinIO/Garage ops overhead (при self-host).
- Netty-transport из AWS SDK заменён на hato-http — единый механизм прокси (по.tight user-требованию) для bot API + S3 + external.