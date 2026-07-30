# ADR A07 — Хранение raw GPX и фото в S3-совместимом хранилище

**Status**: Accepted
**Date**: 2026-07-25

**Уточнение 2026-07-30:** raw GPX private и хранится как exact original upload bytes
без application encryption или преобразования; наружу выдаётся отдельный sanitized
export. Objects используют opaque UUID keys, согласование с БД идёт через upload
jobs. Полный контракт:
[Implementation Contract](../IMPLEMENTATION-CONTRACT.md#12-object-storage-и-upload-jobs).

**Уточнение 2026-07-29:** durable upload-flow lease, generation и slot accounting
хранятся в PostgreSQL и транзакционно согласуются с draft metadata. Valkey содержит
только ephemeral chat controls и не является источником ownership/slot state.

## Контекст

PostGIS хранит распарсенную geometry треков и метаданные. Но нужны durable
original raw objects для re-parse, отдельный sanitized export для пользовательского
download и objects фотографий. PostGIS-geometry — это processed projection, а не
источник истины. БД не подходит для blob-хранилища — твёрдый паттерн
`geometry in DB, objects in S3-compatible storage`.

## Решение

1. **S3-совместимое хранилище** для raw GPX и фото (MinIO / Garage / Cloudflare R2 — self-hosted совместимо).
2. **Доступ из Clojure-бэкенда**: **aws-simple-sign** (SigV4 подпись, zero-dep) + **hato** (transport с прокси) — см. A08. Subiesto одной HTTP-стеке для bot API, S3, и external fetches.
3. **Паттерн**:
   - Upload: exact original raw bytes записываются без application encryption под
     opaque UUID key, затем async job создаёт private revision в PostGIS. Raw
     доступен только backend workers для parse/re-parse; public/owner download route,
     presigned raw URL и HTTP streaming endpoint отсутствуют. После successful parse
     raw не имеет отдельного TTL: он хранится до физической очистки связанного
     draft/track, включая published и archive/appeal retention, и всё время входит в
     user raw-storage quota. Confirmed draft delete или track purge удаляет raw;
     24-hour janitor затрагивает только incomplete/orphan objects.
     Новый upload всегда создаёт новый immutable exact raw object без content-based,
     cross-track или cross-user dedup. Re-parse/metadata revision того же track без
     нового upload переиспользует ссылку на existing object и не копирует source
     bytes; quota считает object один раз, cleanup ждёт последнюю durable reference.
     Отдельная PostgreSQL table `raw_objects` владеет `owner_id`, opaque S3 key,
     exact `byte_size`, private HMAC, lifecycle state и timestamps.
     `upload_jobs.raw_object_id` и
     `track_revisions.raw_object_id` ссылаются на неё; revision snapshots не
     дублируют storage metadata.
     Retained revision всегда pin-ит raw и использует `ON DELETE RESTRICT`.
     Nullable job FK использует `ON DELETE SET NULL` и pin-ит object только пока job
     продолжим либо допускает explicit transient retry до 24-hour incomplete
     deadline. Success передаёт pin revision, cancel/permanent failure снимает его;
     terminal job history cleanup не блокирует.
     Reuse и last-reference cleanup сериализуются locks в порядке
     `users -> raw_objects`, обе rows берутся `FOR UPDATE`. Reuse проверяет
     owner/same-track-lineage/`ready`; cleanup повторяет pin query перед
     `delete_pending`+outbox. `delete_pending` не оживляется, поэтому cleanup-first
     требует нового upload; revision FK остаётся DB safety net.
     Lifecycle содержит только `pending`, `ready`, `delete_pending`. `pending`
     создаётся до PUT, revision может ссылаться только на checksum-validated `ready`,
     а последняя reference removal или 24-hour incomplete janitor атомарно создаёт
     `delete_pending` и cleanup outbox command. Успешный S3 delete удаляет row;
     upload error хранится в job, delete retries/DLQ — в command, поэтому raw
     `error`/`deleted` states отсутствуют.
     Cleanup command несёт только `raw_object_id`. Worker читает key из current
     `delete_pending` row; S3 DELETE success/`404`, missing row при replay и crash
     между S3/DB delete обрабатываются идемпотентно. Wrong state или retained FK
     fail-closed проходят retry/DLQ+alert; storage metadata в outbox не
     копируется.
     Quota считает `pending` как fixed 10 MiB reservation без доверия внешнему size,
     `ready` — по actual bytes, `delete_pending` — как zero с момента
     commit. Terminal upload failure ставит `delete_pending` сразу; 24-hour janitor
     является fallback, а cleanup lag/DLQ не штрафует quota owner-а.
     Cached user counter отсутствует: quota-changing transaction сначала блокирует
     owner `users` row, затем вычисляет indexed SQL sum по `raw_objects` и вставляет
     reservation в той же transaction. Partial covering index охватывает
     `owner_id/state` для `pending|ready` и включает byte size.
   - Publish: private sanitized export pre-generate-ится для immutable pending
     revision. Moderator approve разрешён только при durable `export_state = ready` и
     одной PostgreSQL transaction переключает revision/track в published/current.
     Generation error оставляет `pending_review` и старый current без публичного
     `publishing` status; job проходит retry/DLQ. Moderator видит pending/error badge,
     может review/changes-request/reject и после automatic exhaustion идемпотентно
     requeue-ить export; Approve disabled до ready, owner infra details не видит.
     До approval private sanitized object доступен только backend workers:
     owner/moderator UI не получает presigned URL и не имеет отдельного authenticated
     download route. Moderator видит `export_state` и retry action; download
     появляется только после publication через canonical route.
     Changes-request/reject ставит `export_state = discarded` и high-priority delete;
     новый revision export не переиспользует. Late worker completion не меняет
     discarded state и ставит созданный object на cleanup с retry/DLQ/alert.
     Multilingual Description сериализуется в один standard GPX `<desc>` как
     deterministic `[ru]`, затем `[en]` plain-text blocks; пустое поле отсутствует,
     viewer variants/custom extensions не создаются.
     Provenance/license использует только standard GPX 1.1: root
     `creator="TrailBase revision:<uuid>"`, author display name в
     `<metadata><author><name>`, CC BY 4.0 в
     `<metadata><copyright author="..."><license>`, canonical track URL в
     `<metadata><link href="...">`, а user Name/Description в
     `<trk><name>`/`<trk><desc>`.
     Provider/internal identity, raw filename и application extensions отсутствуют.
     Один export содержит один `<trk>` на revision и один `<trkseg>` на каждый
     canonical `MultiLineString` component в deterministic order. Исходная
     multi-track grouping не восстанавливается; route-only input тоже экспортируется
     как `<trk>`, не `<rte>`.
     `<ele>` включается только для retained point с валидным source elevation;
     missing values не заменяются zero/interpolation и не берутся из
     smoothed/LTTB-профиля. Derived 90% coverage threshold на export отдельных
     available values не влияет.
     Numeric text округляется `HALF_EVEN`: lat/lon максимум 7 decimal places,
     elevation максимум 2; plain decimal notation без exponent/trailing zeros,
     negative zero становится `0`. Одинаковый revision input даёт byte-identical
     numeric text. Весь GPX byte-identical для одной revision: UTF-8 без BOM, fixed
     XML declaration/GPX 1.1 namespaces/element and attribute order, compact XML,
     один final LF и никаких runtime timestamps/random values.
     Durable export hash/size в PostgreSQL не хранится. PUT подписывает actual
     payload SHA-256 и ready наступает только после checksum-validated S3 success;
     byte determinism проверяют golden tests, не public `ETag`.
     `<metadata><bounds>` отсутствует: viewers вычисляют его по points, а отдельный
     min/max bbox дублирует geometry и неоднозначен на antimeridian.
     Activity/difficulty/tags не записываются в `<trk><type>` или keywords:
     TrailBase taxonomy остаётся на canonical track URL/JSON API.
   - Download: anonymous canonical endpoint без account/session до lookup/signing
     применяет 30 attempts/min на normalized IP с burst 10, считает `404` и выдаёт
     presigned URL отдельного sanitized GPX с TTL 5 минут; session limit не повышает,
     redirected S3 GET не считается. Private/archived/moderator-removed state
     отвечает одинаковым `404`, raw не публикуется, bot не раскрывает signed URL.
     Все canonical `302/404/429/5xx` имеют `Cache-Control: no-store`, без CDN
     cache/`ETag`; каждый request заново проверяет limit/publication state.
     Sanitized object metadata задаёт
     `Content-Type: application/gpx+xml; charset=utf-8` и
     `Content-Disposition: attachment` с ASCII slug/UUID filename, поэтому final S3
     response после redirect остаётся download.
     Filename использует NFKD Name, lowercase ASCII alnum, collapsed `-`, max 64 и
     fallback `track`, затем первые 8 lowercase hex stable track UUID и `.gpx`;
     transliteration/raw filename отсутствуют.
     Object и response остаются exact uncompressed XML bytes без `Content-Encoding`,
     gzip/ZIP или второго compressed representation.
     Sanitized object не шифруется приложением: private bucket хранит exact XML,
     presigned HTTPS URL отдаёт его напрямую без backend decrypt proxy. Transparent
     provider-side SSE или disk encryption допустимы как deployment control и не
     меняют object bytes/API.
     Canonical route разрешает
     только current published revision; revision-specific public URL отсутствует,
     старые exports могут очищаться, а новая approval переключает тот же route.
     Supersede/non-public transition ставит high-priority idempotent delete старого
     export с retry/DLQ/alert; выданный URL после delete не работает, а при задержке
     ограничен исходным five-minute TTL. Download proxy не вводится.
   - Фото также используют opaque object keys и private bucket policy.
4. **Re-parse pipeline**: backend читает original raw bytes из S3, парсит их и
   создаёт новую immutable revision с reference на тот же raw object. Source bytes не
   копируются, опубликованный snapshot не изменяется на месте.
5. **Durable upload-flow coordination**: PostgreSQL хранит active lease, generation,
   interface/activity timestamps и user slot state. Fixed `slot_no` 1..3 и partial
   unique indexes на active user/slot и draft обеспечивают concurrency invariants;
   slot lifecycle сериализуется row lock на `users` с порядком `user -> flow -> draft`;
   Valkey loss может инвалидировать только ephemeral interaction tokens.
6. **Cancel coordination**: durable `cancel_requested_at` проверяется перед S3 write,
   parse и revision commit. Cancel-wins не создаёт draft и чистит temporary object;
   commit-wins сохраняет draft/raw по post-parse cancel policy.
7. **Account deactivation during parse**: already-started job может завершить
   validation/cleanup и создать только private draft. Worker не выполняет automatic
   submit/publication; pending moderation отдельно проверяет active owner.
8. **Bot status projection**: каждый `upload_job_id` получает одну status card,
   которая best-effort редактируется на стадиях download/validation/parse. Terminal
   state показывает draft actions либо безопасную ошибку/retry; provider edit failure
   создаёт replacement card, но никогда не повторяет durable domain job.
9. **Explicit retry**: после исчерпания automatic transient attempts тот же
   `upload_job_id` атомарно reacquire-ит user slot и requeue-ится, переиспользуя
   сохранённый raw object. Permanent GPX/limit/geometry error и отсутствующий raw
   требуют новый attachment; второй job для retry не создаётся.
10. **Post-parse chat editing**: metadata заполняются в любом порядке из одной draft
    summary card; prompts bound к конкретным job/lease generation. Name и Activity
    required, submit с пропусками не выполняет mutation, а готовый draft показывает
    final summary и CC BY 4.0 confirmation.
11. **Name и filename fallback**: initial Name выбирается из file-level
    `<metadata><name>`, затем из единственного distinct непустого `<trk><name>`,
    затем из безопасно нормализованного stem attachment filename. Несколько разных
    track names дают warning и не выбираются/не объединяются. Default остаётся
    редактируемым и неподтверждённым; исходный filename отдельно не сохраняется, не
    входит в object key или export и никогда не публикуется. Route-only input
    использует тот же порядок с `<rte><name>`.
12. **Description и attachment caption**: initial Description выбирается из
    file-level `<metadata><desc>`, затем из единственного distinct непустого
    `<trk><desc>`. Несколько разных track descriptions дают warning и пустой optional
    field без выбора/объединения. Caption не копируется в draft/public metadata и
    остаётся только в техническом raw-webhook retention; Description получает
    выбранный GPX default либо explicit text из job-bound prompt. Route-only input
    использует тот же порядок с `<rte><desc>`.
13. **GPX description locale**: `<desc>` без locale сохраняется как неподтверждённый
    default только под текущим saved UI locale owner (`ru`/`en`) на момент upload.
    Autodetect/translation/duplication нет; последующая смена UI locale text не
    переносит.
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
