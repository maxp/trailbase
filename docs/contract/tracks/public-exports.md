# Контракт: public GPX exports и licenses

Часть [Implementation Contract](../../IMPLEMENTATION-CONTRACT.md). При противоречии с другими документами действует этот контракт.

## 14. Public export и лицензии

- Original raw GPX никогда не публикуется и не выдаётся owner-у отдельным private
  download. Для raw отсутствуют HTTP streaming endpoint и presigned delivery; он
  доступен только backend workers как внутренний parse/re-parse source.
- Sanitized GPX pre-generate-ится приватно для immutable pending revision и хранится
  отдельно. Он содержит только lat/lon/elevation, submitted name/description и
  license metadata; public доступ появляется лишь после approve transaction при
  `export_state = ready`.
- На application layer sanitized object является незашифрованным exact XML в private
  bucket. TrailBase выдаёт его напрямую по five-minute presigned HTTPS URL без
  decrypt proxy; прозрачное provider-side SSE/disk encryption не меняет этот
  контракт.
- До approval отсутствуют owner/moderator download endpoint и presigned delivery
  private sanitized object; доступ к object имеет только backend. Public download
  возникает только после approve через canonical current-revision route.
- Timestamps, heart rate, cadence, device extensions и original waypoints удаляются.
- Provenance и license записываются только в стандартные поля GPX 1.1, без
  application extensions: root `creator="TrailBase revision:<uuid>"`;
  `<metadata><author><name>` содержит TrailBase author display name;
  `<metadata><copyright author="..."><license>` содержит canonical
  `https://creativecommons.org/licenses/by/4.0/`; `<metadata><link href="...">`
  содержит canonical track URL. Пользовательские Name и Description остаются в
  `<trk><name>` и `<trk><desc>`. Provider identity, internal user ID, raw filename и
  application extensions отсутствуют.
- Каждый sanitized GPX содержит ровно один `<trk>` на TrailBase revision. В нём
  находятся один submitted `<name>` и один сформированный `<desc>`, а каждый
  component canonical `MultiLineString` сериализуется отдельным `<trkseg>` в
  deterministic component order. Исходная grouping нескольких `<trk>` не
  восстанавливается; route-only input также экспортируется как `<trk>`, не `<rte>`,
  поскольку публичная единица каталога — один TrailBase track.
- `<trkpt><ele>` создаётся только для сохранившейся после trimming point с валидным
  исходным elevation. При отсутствии значения `<ele>` у этой point отсутствует:
  export не интерполирует elevation, не подставляет zero и не использует
  smoothed/LTTB profile. Правило одинаково для track/route input и не зависит от 90%
  coverage threshold, который управляет только derived profile и gain/loss.
- Numeric text export-а детерминирован: lat/lon округляются `HALF_EVEN` максимум до
  7 знаков после decimal point, `<ele>` — максимум до 2. Используются decimal dot и
  plain notation без exponent; незначащие trailing zeros удаляются, negative zero
  нормализуется в `0`. Одинаковые revision inputs дают byte-identical numeric text.
- Весь serialized GPX для одной revision byte-identical: UTF-8 без BOM, фиксированная
  XML declaration `<?xml version="1.0" encoding="UTF-8"?>`, фиксированные GPX 1.1
  namespace declarations и schema-conforming element/attribute order. Serializer
  пишет compact XML без insignificant whitespace и ровно один LF в конце. Generation
  timestamps, random values и иные runtime-dependent bytes отсутствуют.
- Durable SHA-256 и byte-size поля export-а в PostgreSQL отсутствуют. S3 PUT
  подписывает actual payload SHA-256, не `UNSIGNED-PAYLOAD`, и `export_state` может
  стать `ready` только после успешной checksum-validated записи. Byte determinism
  проверяется serializer golden tests; hash/size не публикуются как HTTP `ETag` и не
  меняют `no-store` canonical download route.
- `<metadata><bounds>` в sanitized GPX отсутствует. Viewers вычисляют bounds из
  `<trkpt>`; derived element дублировал бы geometry, а обычный min/max longitude
  неоднозначен для antimeridian tracks.
- TrailBase activity, difficulty и tags не сериализуются в `<trk><type>` или
  `<metadata><keywords>`. Эти GPX-поля не имеют interoperable vocabulary для
  TrailBase taxonomy; classification остаётся доступна по canonical track URL и в
  JSON API без отдельного versioned GPX mapping.
- Standard GPX `<desc>` export-а является deterministic plain text: каждая непустая
  description branch включается с явной меткой `[ru]`/`[en]` в фиксированном порядке
  `ru`, затем `en`, блоки разделяются пустой строкой. Если обе ветви пусты, `<desc>`
  отсутствует. Viewer-specific variants, machine translation и custom XML extensions
  не используются; значения проходят обычный XML escaping.
- Author display name в уже сгенерированном sanitized GPX является immutable частью
  export object и не переписывается массово при смене `users.display_name`. Новый
  export object, включая export новой approved revision, использует актуальное имя на
  момент генерации.
- Автоматическое agreement первого `/start` действует как standing consent на
  публикацию contributions под CC BY 4.0 и допускает последующие изменения правил.
  Каждая отправка на moderation показывает reminder и canonical license link, но не
  требует отдельного acceptance.
- Каталог как база данных публикуется под ODbL 1.0. Перед первым bulk export требуется
  юридическая проверка текста attribution.
- Anonymous public download sanitized GPX не требует account/browser session и не
  создаёт identity или session. Canonical download endpoint повторно проверяет
  publication status, применяет до lookup/signing лимит 30 запросов/мин на normalized
  client IP с burst 10 и отвечает `302` на presigned URL с TTL 5 минут. Все попытки,
  включая `404`, расходуют лимит; session его не повышает, S3 GET после redirect не
  учитывается. Private, archived и moderator-removed revision получают одинаковый
  `404`. Otherwise published track под active full track lock получает generic `503`
  без `Retry-After`, track metadata или integrity details; bot публикует canonical
  track/download URL, а не presigned URL.
- Все ответы canonical download endpoint (`302`, `404`, `429`, `5xx`) содержат
  `Cache-Control: no-store`; route не использует CDN cache или `ETag`. Каждый request
  заново проходит limiter и publication check, поэтому cached redirect не обходит
  revision switch/removal. Presigned S3 URL остаётся отдельной delivery-ссылкой с
  собственным five-minute expiry.
- Активация full track lock прекращает выдачу новых presigned URLs после того, как
  canonical request увидел active lock, но не удаляет sanitized object. URL, выданный
  до lock, остаётся валиден не более своего исходного five-minute TTL; отдельная
  rotation/revocation mechanism для него не добавляется.
- `/tracks/:track-id/download` разрешает только current published revision.
  Revision-specific public download endpoint в MVP отсутствует; sanitized exports
  старых revisions не адресуются публично и могут очищаться по storage retention.
  Commit нового approved `current_revision_id` автоматически переключает тот же
  canonical download URL. Старой ссылкой нельзя запросить superseded moderation
  snapshot, geometry до privacy trim или removed track.
- Любой переход export из current/public в superseded или non-public пишет
  high-priority idempotent delete command старого sanitized object, кроме описанного
  90-дневного appeal retention последнего approved export после moderator removal.
  S3 delete использует retry/DLQ и operator alert; proxy-download для мгновенного
  revoke не добавляется. После успешного delete ранее выданный presigned URL перестаёт
  работать; при временной задержке остаточный доступ ограничен его исходным TTL не
  более пяти минут.
- Sanitized S3 object хранит response metadata
  `Content-Type: application/gpx+xml; charset=utf-8` и
  `Content-Disposition: attachment; filename="<ascii-name-slug>-<uuid-fragment>.gpx"`.
  Поэтому headers применяются к final S3 response после presigned redirect; inline
  rendering и отдельный UTF-8 filename fallback не используются.
- S3 хранит и отдаёт exact uncompressed XML bytes. `Content-Encoding` отсутствует;
  gzip, ZIP и второе compressed representation в MVP не создаются. Canonical
  attachment всегда имеет расширение `.gpx`.
- Filename строится из final submitted revision Name без transliteration dependency:
  Unicode NFKD, lowercase ASCII letters/digits сохраняются, каждый run остальных
  codepoints становится одним `-`, края trim-ятся. Slug обрезается до 64 символов с
  повторным trailing-`-` trim; пустой slug становится `track`. Затем добавляются `-`,
  первые 8 lowercase hex символов stable track UUID и `.gpx`. Исходный upload
  filename не используется; полный Unicode Name остаётся внутри GPX.
