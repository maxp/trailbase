# TrailBase — Architecture

TrailBase — открытый каталог GPX-треков с отображением на карте, поиском, классификацией и газеттиром известных локаций (POI). Доступ через веб-сайт (first-class) и ботов Telegram/Max.

Масштаб: **публичный каталог сообщества** — десятки тысяч треков, тысячи пользователей, открытая загрузка с модерацией.

Практические инварианты реализации, принятые после ADR-серии, собраны в
[Implementation Contract](IMPLEMENTATION-CONTRACT.md). При расхождении краткого
обзора или roadmap с этим контрактом действует контракт.

## Зафиксированные решения

| Домен | Решение | ADR |
|---|---|---|
| Масштаб и владение | Публичный каталог, multi-user, модерация | — |
| БД | PostgreSQL + PostGIS | A05 |
| Хранение GPX | Private S3-compatible storage; exact original raw и sanitized export | A07 |
| Поиск | Единственный engine: PostgreSQL (tsvector+GIN мультиязычный, GIST, btree, JOIN через POI) | A05 |
| Auth | Telegram/Max identity достаточна для chat access; optional one-time token → Valkey-backed web session; email/phone отсутствуют | A01 |
| GPX-доставка | Exact original raw в S3 → async parse → 2D MultiLineString revision в PostGIS + sanitized export | A02 |
| Доставка карты | Готовые OSM raster-тайлы (без генерации тайлов) + adaptive GeoJSON по bbox + zoom-aware simplification | A02 |
| Фронтенд | htmx + alpine.js (server-rendered partials) + MapLibre GL JS как остров с bridge glue | A03 |
| Боты | First-class chat UI: identity, upload, search и уведомления поверх общих domain services | A01 |
| Классификация | Иерархическая таксономия: основной тип активности (enum) + вторичные теги (moderated vocabulary) | — |
| Структурные фасеты | difficulty (по стандарту активности: SAC/MTB-Scale/3-level), season (bitmask), duration (auto из GPX или manual, `duration_source`) | A06 |
| POI / газеттир | Гибрид: самостоятельные сущности Point/Line/Area + OSM-provenance кэш, наполнение модератором/заявкой | A04 |
| Рендеринг треков на клиенте | Adaptive zoom: low-zoom кластер POI (server-side clustering в PostGIS), mid/high-zoom упрощённые полилинии | A02 |
| Backend язык | Clojure | A08 |
| Clojure-стек | reitit+ring, next.jdbc, honeysql+hugsql, Malli, Hiccup, migratus, data.xml, hato+proxy, aws-simple-sign, jsonista, telemere, deps.edn+babashka | A08 |

## Домен

### Треки
- `tracks` хранит stable identity и `current_revision_id`; versioned geometry,
  метаданные, фасеты и tags находятся в immutable `track_revisions`.
- Canonical geometry — 2D MultiLineString. Один GPX attachment создаёт один draft;
  каждый валидный `<trkseg>` остаётся отдельным component в document order.
  Segment manifest отдельно не хранится; исходная hierarchy остаётся в original raw
  GPX. Для multi-track/route warning сохраняются только
  source-track/source-route/valid-segment counts; отдельного confirmation gate до
  final submit нет. Route-only attachment использует те же one-draft и
  distinct-value metadata rules. Elevation profile хранится отдельно.
- Auto-derived из GPX при загрузке: length, elevation gain/loss, duration (если есть `<time>`), activity-предсказание как подсказка.
- Canonical duration по умолчанию использует GPX moving, затем elapsed, затем unknown;
  manual/unknown override не удаляет рассчитанные moving/elapsed values. Manual value
  ограничен целыми секундами от 1 секунды до 365 дней; unknown — `NULL`, не zero.
  Outlier warning сравнивает manual с тем же moving/elapsed default candidate.
- Duration outlier acknowledgement хранится в PostgreSQL и привязано к exact manual
  и comparison inputs; Valkey содержит только одноразовый chat control.
- Moderator видит manual/auto duration comparison и acknowledgement как
  informational flag без automatic reject или queue reprioritization.
- Moderator не редактирует pending revision: author correction после
  `changes_requested` создаёт новый immutable snapshot с повторной validation.
- Submit на moderation требует подтверждённый final name и явно выбранный primary
  activity; description optional, auto-derived metrics подтверждаются общим final
  summary.
- Автоматическое agreement первого `/start` — standing CC BY 4.0 consent. Revision
  submit показывает reminder/link и не имеет отдельного agreement gate.
- Pre-computed упрощённые геометрии: 3 уровня (z11/z13/z15) в `geometry_simplified_*`, on-the-fly fallback.
- discovery: bbox-filtered, zoom-aware simplification; NOT все треки в низких zoom'ах.

### POI (газеттир)
- `locations` имеет stable identity и immutable revisions. Geometry kind
  (`point/line/area`) отделён от semantic category.
- POI links — модерируемые annotations конкретной geometry revision; одна связь может
  иметь несколько ordered occurrences.
- High-confidence autodetect approved locations публикуется автоматически, остальные
  совпадения идут в moderation.
- Low-zoom использует deterministic server-side hex-grid clustering в PostGIS.

### Поиск
- Единственный backend — PostgreSQL.
- Текст: `tsvector` мультиязычный (russian/english/simple) + GIN + триграммы для fuzzy.
- Гео: GIST-индекс, ST_Intersects/ST_DWithin.
- Фасеты: compound btree indexes.
- POI-join: JOIN через approved revision-location annotations.
- Instant-search (<200ms target), все facet counts считаются server-side с
  disjunctive semantics.

### Auth
- Telegram/Max — единственные identity providers; email/phone/recovery отсутствуют.
- Валидированная messenger identity достаточна для account и разрешённых chat
  operations; browser activation не требуется.
- Первый валидированный `/start` неизвестной identity в private one-to-one chat
  атомарно создаёт active account и provider identity; повторная доставка идемпотентно
  использует тот же account. Identity и token flows в группах/каналах запрещены.
- Первый plain `/start` в той же transaction создаёт минимальный `user_agreements`
  record. Ответ показывает описание, главное меню, rules/CC BY 4.0 links и сообщает,
  что использование бота означает автоматическое согласие; кнопки «Согласен» и
  pending state нет. Linking `/start <token>` остаётся отдельным fail-closed flow.
- Главное меню Telegram/Max одинаково и содержит «Поиск», «Загрузить GPX», «Мои
  треки/черновики», «Настройки» и «Помощь». Rules/license остаются вторичными
  ссылками.
- «Настройки» разделены на «Профиль», «Мессенджеры», «Уведомления», «Сессии» и
  «Аккаунт». Email/phone/password/recovery settings отсутствуют.
- Все settings operations завершаются в private chat без web session, включая
  session list/revoke/logout-all и deactivation с принятыми fresh-auth/confirmation
  guards. Web settings UI остаётся optional mirror.
- Chat session card содержит только short device/browser summary, created/last-seen/
  expiry и revoke action; IP/full UA/session identifiers отсутствуют, callback
  использует bound opaque state.
- Single-session revoke требует short confirmation без fresh auth и атомарно
  re-check-ит owner/target; logout-all остаётся отдельной fresh-auth operation.
- `logout-all` после fresh-auth confirmation атомарно revoke-ит все web sessions,
  не затрагивая messenger identities, chat access или account lifecycle.
- Public display name обновляется сразу без fresh auth/pre-moderation после
  NFC/trim/length validation; изменение audit-ится, provider snapshots его не
  перезаписывают.
- Public track pages показывают текущее account display name даже для старых
  published revisions. Generated GPX export остаётся immutable с именем на момент
  генерации; новый export использует актуальное имя.
- User-selectable UI locales — `ru` и `en`, общие для bot/web и устойчивые к provider
  profile updates. `simple` — только technical content/search fallback.
- Security и action-required moderation notifications (`changes_requested`,
  `rejected`) locked-on; catalog/informational/other moderation delivery configurable.
- Deactivated account получает только locked-on security/account-lifecycle
  notifications через web inbox и primary messenger. Все moderation, catalog и
  informational notifications подавляются без последующего replay; domain
  events/audit сохраняются.
- Deactivation отменяет pending non-security notification outbox delivery intents и
  suppress-ит связанные unread inbox records с сохранением retention. Уже claimed
  `sending` delivery может завершиться; security/account-lifecycle delivery не
  отменяется.
- Defaults нового account: other moderation results enabled,
  catalog/informational disabled.
- Optional notification preference account-level и едино для web inbox/primary bot;
  domain state/events/audit не зависят от него. Per-channel toggles отсутствуют.
- Preference snapshot фиксируется в notification transaction; dispatcher не
  перечитывает settings, а изменения действуют только на future events. Deactivation
  suppression — отдельное lifecycle exception.
- «Мои треки/черновики» без web session объединяет owned upload/draft flows и tracks,
  показывает `draft`, `processing`, `pending_review`, `changes_requested`,
  `published` или `rejected` и сначала выводит требующие действия пользователя.
- User-archived tracks вынесены во вторичный filter «Архив» с оставшимся сроком до
  30-дневного purge и действием restore; early permanent delete в MVP отсутствует.
- List views ограничены «Все» и «Архив»; дополнительных status filters в MVP нет.
- User archive durable запоминает pre-archive status/current revision. Restore того же
  track до purge возвращает их атомарно; прежний approved snapshot не проходит
  moderation повторно. Moderator removal этим flow не отменяется.
- Restore выполняется одним valid owner callback без confirmation prompt; domain
  mutation проверяет `archived`/`purge_at` атомарно и идемпотентна при повторной
  доставке.
- Карточки дают status-dependent actions: edit для `draft`/`changes_requested`,
  progress/cancel для `processing`, read-only для `pending_review`,
  open/download/new revision/archive для `published`, reason/new revision для
  `rejected`. Delete/archive требуют отдельного confirmation.
- Список листается по 10 entries через «Назад»/«Далее». Keyset cursor и
  provider/chat/message/requester binding находятся в Valkey; callback несёт только
  opaque ID.
- List controls имеют абсолютный TTL 15 минут без продления. Expiry или потеря Valkey
  оставляет старое сообщение неизменным и требует открыть список заново.
- Bot показывает обычные HTTPS-ссылки на актуальные rules и CC BY 4.0; документы не
  копируются в chat. Изменения rules по ссылке не версионируются в account model и
  никак не влияют на agreement, account, sessions или notifications.
- Append-only `user_agreements` в PostgreSQL хранит только user ID, notice hash,
  rules/license URLs, acceptance timestamp и `/start` source; unique user исключает
  дубликаты. IP, raw tokens и callback state отсутствуют.
- Повторный plain `/start` не меняет agreement record; обновляется только provider
  profile snapshot, notice/menu показываются снова.
- Деактивация и административная реактивация сохраняют исходный agreement record и
  не требуют нового agreement; физическое удаление регулируется отдельной policy вне
  MVP.
- Published tracks деактивированного account остаются public с прежним TrailBase
  author display name и CC BY 4.0 attribution. Новые login/mutations блокируются,
  private content остаётся private, а скрытие/удаление track является отдельной
  lifecycle operation.
- Self-deactivation после fresh auth собирает только explicit confirmation без
  reason/free text/feedback. Audit содержит actor=user, action, interface/provider,
  request ID и timestamp.
- Final confirmation без dynamic counts перечисляет отзыв sessions/tokens,
  блокировку account operations, сохранение public CC BY 4.0/private content,
  private-only parse completion, removal pending review из queue и возврат только
  через support/admin. Actions: «Деактивировать аккаунт»/«Отмена».
- Confirmation истекает в original `fresh_authenticated_at + 10 minutes` без
  продления. Chat использует bound 128-bit opaque ID, web — active session/CSRF и
  bound purpose state; Confirm/Cancel single-use, invalid/stale/replay fail-closed.
- State находится в PostgreSQL `sensitive_operation_confirmations` как SHA-256 opaque
  ID и bindings/timestamps. Confirm под row lock атомарно consume-ит row,
  деактивирует account и пишет audit/outbox; Cancel только consume-ит row. Raw ID не
  логируется, terminal/expired rows очищаются через 24 часа.
- Session validation и auth/link token consume всегда проверяют PostgreSQL active
  status. Deactivation transaction пишет idempotent credential-cleanup outbox command
  без raw tokens; worker удаляет Valkey keys с retry/DLQ. Cleanup lag/failure не
  возвращает доступ и alert-ит operator.
- Дополнительного credential epoch/version нет: deactivation flow очищает account
  sessions и outstanding auth/link tokens непосредственно в Valkey через этот
  cleanup command.
- При недоступном PostgreSQL authorization работает fail-closed без cached-active
  fallback: account-specific HTTP получает `503` с `Retry-After` без
  session/mutation, bot event проходит transient retry/DLQ без domain changes, а
  Valkey credential сам по себе не даёт доступ.
- Session-validation `503` не очищает cookie/Valkey session и не продлевает
  `last_seen_at`/sliding TTL; ответ имеет `Cache-Control: no-store`. После
  восстановления PostgreSQL та же неистёкшая session снова работает; revoke
  применяется только к подтверждённому invalid, expired или disabled состоянию.
- Public catalog не раскрывает деактивацию автора badge, причиной или иным
  account-status marker; status доступен только самому пользователю через нейтральный
  auth response и администраторам через management/audit.
- In-flight parse deactivated owner может завершиться только private draft.
  `pending_review` сохраняет status без отдельного `suspended`, исключается из
  moderator queue и снова становится eligible после реактивации; approve/publish
  fail-closed проверяет active owner.
- Deactivation и approve/publish сериализуются owner row lock с порядком
  `user -> track -> revision`; первый commit побеждает. Approval-first публикация
  остаётся public, deactivation-first блокирует публикацию.
- Linked disabled identity всегда resolve-ится в прежний account и получает только
  `/start`, help/rules/license и stateless read-only public search. Account-specific
  data, upload/settings, auth/link tokens и mutations недоступны; новый account не
  создаётся.
- Встроенного bot reactivation-request workflow нет. Disabled `/start` показывает
  обязательный production `TRAILBASE_SUPPORT_URL`; после out-of-band проверки admin
  реактивирует account в management UI с fresh auth, обязательной причиной и audit,
  создавая locked-on lifecycle notification.
- Reactivation reason — required enum `support_request_verified`,
  `administrative_correction`, `other`; audit note optional, кроме обязательной
  validated note для `other` до 1 000 Unicode code points. Code/note хранятся только
  в append-only audit и не попадают в notification/logs.
- Reactivation возвращает тот же account в active с сохранёнными identities, roles,
  settings, private content и pending moderation. Sessions, auth/link tokens и
  suppressed notifications не восстанавливаются; agreement не меняется, lifecycle
  notification не содержит token, а web login начинается отдельно.
- Reactivation notification показывает только факт/time, `TRAILBASE_SUPPORT_URL` и
  инструкцию открыть bot; admin identity, internal reason, audit ID и tokens остаются
  скрыты, reason хранится только в audit.
- Optional web entry: bot выдаёт 10-minute Valkey token → safe GET/POST confirmation →
  Valkey session cookie.
- Единый аккаунт с explicit provider linking: `/start <link-token>` второго provider
  добавляет identity к target account без нового account и browser completion;
  автоматического account merge нет.
- Agreement принадлежит `user_id`, поэтому linked identity сразу использует
  capabilities target account без новой agreement записи.
- Session cookie содержит 128-bit opaque token; sliding TTL — один год.
- Telegram/Max chats поддерживают upload и search без web session, используя те же
  domain services, permission checks и async pipelines, что web/API.
- Любой GPX attachment в private chat после quota/three-slot check автоматически
  начинает upload flow; `/upload` только предлагает прислать файл и не создаёт
  pending mode. Attachment другого типа job не создаёт. Group/channel `/search`
  работает без account state и видит только public published catalog.
- Каждый bot upload имеет одну status card на `upload_job_id`, best-effort
  редактируемую при download/validation/parse. Terminal state показывает draft
  actions либо безопасную ошибку/retry; provider edit failure создаёт replacement
  card без повторного domain job.
- Explicit retry после исчерпания transient attempts повторно занимает slot и
  requeue-ит тот же job с сохранённым raw object. Permanent file error или
  отсутствующий raw требуют новый attachment; concurrent retry идемпотентен.
- После parse одна editable draft summary card позволяет заполнять metadata в любом
  порядке через job-bound prompts. Name/Activity required; incomplete submit ничего
  не меняет, complete draft показывает final summary и CC BY 4.0 confirmation.
- Name default берётся из file-level `<metadata><name>`, затем из единственного
  distinct непустого `<trk><name>`, затем из безопасно нормализованного attachment
  filename stem. Несколько разных track names дают warning и не выбираются/не
  склеиваются; default требует final confirmation. Raw filename отдельно не
  хранится/публикуется. Route-only input подставляет `<rte><name>` на element-level
  шаге.
- Description default берётся из file-level `<metadata><desc>`, затем из
  единственного distinct непустого `<trk><desc>`. Несколько разных track descriptions
  дают warning и пустой optional field без выбора/склеивания. Attachment caption не
  копируется в draft/public Description и остаётся только в raw-webhook retention;
  explicit job-bound prompt может задать текст вручную. Route-only input подставляет
  `<rte><desc>` на element-level шаге.
- GPX `<desc>` без locale сохраняется только в description branch текущего saved UI
  locale owner на момент upload. Autodetect/translation/duplication нет; смена UI
  locale позже существующий text не переносит.
- Web/bot presentation показывает Description в requested locale или fallback на
  вторую ветвь с language label; без текста секция отсутствует. JSON API возвращает
  полный `ru`/`en` map без machine translation.
- Sanitized GPX использует один deterministic plain-text `<desc>`: непустые `[ru]`,
  затем `[en]` blocks; без descriptions element отсутствует. Viewer-specific export
  variants и custom XML extensions не создаются.
- Один sanitized GPX содержит один `<trk>` на revision и один `<trkseg>` на каждый
  component canonical `MultiLineString` в deterministic order. Исходная multi-track
  grouping не восстанавливается; route-only input также экспортируется как `<trk>`.
- Exported `<ele>` существует только для retained point с валидным source elevation.
  Missing values не интерполируются, не становятся zero и не берутся из
  smoothed/LTTB profile; derived 90% coverage threshold это правило не меняет.
- Numeric GPX text детерминирован: `HALF_EVEN`, до 7 decimal places для lat/lon и до
  2 для elevation, plain notation с decimal dot без exponent/trailing zeros;
  negative zero нормализуется в `0`.
- Весь GPX byte-identical для одной revision: UTF-8 без BOM, fixed XML
  declaration/namespaces/order, compact whitespace, один final LF и никаких
  runtime-dependent timestamps или random values.
- Durable export SHA-256/size в PostgreSQL отсутствуют. S3 PUT подписывает actual
  payload hash, ready требует checksum-validated success, а deterministic serializer
  проверяется golden tests; canonical HTTP `ETag` из этого не появляется.
- Sanitized GPX не содержит `<metadata><bounds>`: viewer вычисляет bounds по points,
  без отдельной antimeridian convention.
- Activity, difficulty и tags не экспортируются через `<trk><type>` или
  `<metadata><keywords>`; canonical taxonomy остаётся на track page/JSON API.
- GPX provenance/license остаётся в standard GPX 1.1: root
  `creator="TrailBase revision:<uuid>"`, author display name в
  `<metadata><author><name>`, CC BY 4.0 в
  `<metadata><copyright author="..."><license>`, canonical track URL в
  `<metadata><link href="...">`, user Name/Description в
  `<trk><name>`/`<trk><desc>`.
  Provider/internal identity, raw filename и application extensions отсутствуют.
- Durable upload lease/generation/slot state находится в PostgreSQL; Valkey хранит
  только ephemeral chat prompts/controls.
- Interactive search callback несёт только opaque ID; 15-минутные binding/query/cursor
  records находятся в Valkey и не восстанавливаются после потери.
- Public sanitized GPX download анонимен и не создаёт account/session. Canonical URL
  до publication lookup/signing применяет 30 attempts/min на normalized IP, burst 10,
  считает `404` и делает `302` на 5-minute presigned URL. Session limit не повышает,
  S3 GET не считается; private/archived/moderator-removed state получает одинаковый
  `404`, а bot никогда не публикует signed URL.
- Sanitized S3 object возвращает `application/gpx+xml; charset=utf-8` и attachment
  disposition с ASCII slug/UUID filename после presigned redirect.
- Object/response содержит exact uncompressed XML bytes без `Content-Encoding`,
  gzip/ZIP или альтернативного compressed representation.
- Sanitized object не шифруется приложением и остаётся в private bucket. Presigned
  HTTPS URL отдаёт exact XML напрямую без decrypt proxy; provider-side SSE/disk
  encryption может быть включено прозрачно как deployment control.
- Raw S3 object содержит exact original upload bytes без application encryption или
  преобразования. Data/master keys, crypto envelope, keyring, rewrap и decrypt path
  для raw отсутствуют.
- Raw доступен только backend workers для parse/re-parse. Public/owner download
  route, presigned raw URL и HTTP streaming endpoint отсутствуют;
  user-facing download всегда использует опубликованный sanitized GPX.
- После successful parse raw не имеет отдельного TTL, хранится до физической очистки
  связанного draft/track во всех lifecycle states и учитывается в user raw-storage
  quota. Confirmed draft delete/track purge удаляет raw; 24-hour janitor чистит
  только incomplete/orphan objects.
- Новый upload всегда создаёт отдельный immutable exact raw object без
  content-based/cross-track/cross-user dedup. Re-parse или metadata revision того же
  track без upload переиспользует existing object; quota считает его один раз, а
  cleanup ждёт удаления последней durable reference.
- `raw_objects` является единственной PostgreSQL entity для owner, opaque S3 key,
  exact `byte_size`, private HMAC, lifecycle state и timestamps. `upload_jobs` и
  `track_revisions` ссылаются через `raw_object_id`, не копируя storage metadata.
- Retained revision всегда pin-ит raw (`ON DELETE RESTRICT`). Nullable job FK
  (`ON DELETE SET NULL`) pin-ит только продолжимый upload/parse или explicit
  transient retry до 24-hour deadline. Success передаёт pin revision,
  cancel/permanent failure снимает его; terminal job history raw cleanup не
  блокирует.
- Reuse и last-reference cleanup сериализуются `users -> raw_objects FOR UPDATE`.
  Reuse проверяет owner/same track lineage/`ready`, cleanup повторяет pin query перед
  `delete_pending`+outbox. Cleanup-first необратим и требует нового upload; revision
  FK защищает physical row delete.
- Raw lifecycle: `pending` до PUT, checksum-validated `ready` как единственное
  revision-referenceable state, затем `delete_pending` вместе с cleanup outbox
  command и physical row delete после S3 success. Abandoned `pending` уходит прямо в
  `delete_pending`; upload error живёт в job, delete retry/DLQ — в command, без raw
  `error`/`deleted` states.
- Raw cleanup command содержит только `raw_object_id`. Worker читает key из
  `delete_pending` row, считает S3 DELETE success/`404` и missing row успешным replay,
  затем удаляет row. Wrong state/FK fail-closed alert-ится; key/HMAC в outbox
  не копируются.
- Quota contribution равен 10 MiB для `pending`, actual byte size для `ready` и
  zero для `delete_pending`. Reservation создаётся атомарно до PUT без доверия
  provider/HTTP size; terminal failure освобождает logical quota через
  `delete_pending`, а S3 cleanup lag/DLQ остаётся operational leak.
- Quota-changing transaction сначала блокирует owner `users` row, затем считает
  indexed SQL sum по `raw_objects` и вставляет reservation атомарно. Partial covering
  index охватывает `owner_id/state` для `pending|ready` и byte size; user
  counter/reconciliation отсутствуют.
- Filename: NFKD final Name → lowercase ASCII alnum/hyphen slug до 64 или `track` →
  первые 8 lowercase hex stable track UUID → `.gpx`; transliteration/raw filename не
  используются.
- Canonical download `302/404/429/5xx` использует `Cache-Control: no-store`, без
  `ETag`/CDN cache; каждый request заново проверяет limiter и publication state.
- Canonical download разрешает только current published revision. Public
  revision-specific routes отсутствуют; superseded exports не публичны и могут
  очищаться, а новая approved revision переключает тот же stable track URL.
- Supersede/non-public transition ставит high-priority idempotent delete старого
  sanitized object с retry/DLQ/alert. Signed URL прекращает работу после delete, а
  при задержке живёт максимум исходные пять минут; download proxy отсутствует.
- Private sanitized export pre-generate-ится для immutable pending revision.
  Approve требует durable `export_state = ready` и атомарно переключает
  published/current pointers; export error оставляет `pending_review` и старый
  current public, без отдельного public `publishing` status.
- До approval object доступен только backend workers. Owner/moderator UI не выдаёт
  presigned URL и не имеет private download route; moderator видит `export_state` и
  retry action. Download появляется только после publication через canonical route.
- Moderator видит pending/error export badge и может review,
  `changes_requested`/reject; Approve disabled до ready. После automatic retries
  доступен idempotent manual export retry и operator alert, а owner видит только
  обычный `pending_review`.
- Changes-request/reject атомарно переводит private export в `discarded` и ставит
  object delete. Новый revision создаёт свой export; late worker completion не
  оживляет discarded state и чистит созданный object через retry/DLQ/alert.

## Стек доставок (dataflow)

```
[Telegram/Max chat]──────── identity/upload/search ────────┐
       │ optional web-session token                       │
       ▼                                                  ▼
[deep-link URL]──▶[Web (htmx+alpine+MapLibre)]──▶[Clojure backend]
                                                        │
                                   ┌────────────┬────────┼────────┐
                                   ▼            ▼        ▼        ▼
                              [PostGIS]      [S3]    [Valkey] [Bot API]
                              revisions/ private    sessions/ responses/
                              geometry   raw/export streams    notifications
```

## Срезы (tracer bullets) — см. ROADMAP

Декомпозиция на вертикально доставляемые срезы — см. [Roadmap](ROADMAP.md).
