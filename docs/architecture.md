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
| Ephemeral state | Valkey 9.x minimum; Streams, sessions/tokens и hash-field expiry | A08 |
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
- Update revision хранит nullable `base_revision_id`: `NULL` для first publication,
  обязательную same-track current revision для изменения published track. Composite
  FK закрепляет lineage, self-reference запрещён. Diff использует эту baseline;
  approval под track lock требует её совпадения с `tracks.current_revision_id`, иначе
  revision stale без mutation/audit или automatic rebase.
- Nullable `correction_of_revision_id` обязателен для авторского resubmit после
  `changes_requested`: он указывает на непосредственно возвращённую immutable
  revision того же track, а resubmit сохраняет её `base_revision_id`. Для остальных
  revisions correction pointer равен `NULL`. Composite same-track FK закрепляет
  lineage, self-reference запрещён.
- При submit immutable revision получает неизменяемый
  `submitted_for_review_at timestamptz NOT NULL`; это единственный age key обычной
  moderation queue.
- Track имеет durable problem marker для storage и других data-integrity нарушений.
  Admin management UI показывает точный reason code и безопасное пояснение; owner в
  web/private Telegram/Max видит только нейтральный факт проблемы с записью.
- Несколько проблем хранятся в PostgreSQL `track_issues` отдельными rows с
  `code text NOT NULL`, `subject_type text NOT NULL`/`subject_id UUID NOT NULL`,
  admin-only `detail text NOT NULL`, `detected_at`, `last_seen_at` и `resolved_at`.
  DB CHECK ограничивает detail 1–1000 символами; application использует только
  code-specific безопасные шаблоны без raw exceptions, HTTP headers/provider body,
  GPX content или credentials. Detail не использует `jsonb`, а машинная семантика
  остаётся в `code`. Закрытый каталог разрешает только whitelisted mismatch type и
  точно известные expected/observed byte counts; oversized read сообщает lower
  bound. IDs/storage names/digests/provider data/exceptions/operator free text не
  подставляются, а ручные пояснения остаются audit notes.
  Rendered detail всегда canonical English, не зависит от `ru`/`en` UI locale и не
  переписывается; admin UI локализует labels/actions вокруг stored diagnostic text.
  `code` имеет DB CHECK только на `raw_object_missing`, `raw_integrity_mismatch`,
  `sanitized_export_missing` и `snapshot_integrity_unknown`, без PostgreSQL enum;
  новый code требует migration CHECK вместе с checker, subject constraint и
  mapping. Отдельной scope column нет: blocked
  capabilities выводятся из закрытого code mapping:
  raw missing/integrity mismatch → reparse, sanitized export missing →
  download/publication switch, snapshot integrity unknown → full track lock. Subject
  type имеет DB CHECK `track|raw_object|revision`, без PostgreSQL enum; track-wide
  subject повторяет `track_id`, raw использует `raw_objects.id`, revision/export —
  `track_revisions.id`. Compound CHECK разрешает raw codes только с `raw_object`, а
  export/snapshot codes — только с `revision`; начальный code set не допускает
  `track`, пока первый track-wide code не расширит code CHECK и pair CHECK
  миграцией. Partial unique active identity без NULL/sentinel semantics
  предотвращает duplicates;
  `has_problems` — derived flag наличия active rows, owner не получает codes/details.
- `track_id` имеет FK; polymorphic `subject_id` — historical UUID без FK, retention
  pin или storage-reference semantics. Checker под track lock проверяет subject
  membership; DB CHECK для `track` требует `subject_id = track_id`. Purge
  raw/revision не блокируется issue history; physical purge track удаляет все
  active/resolved issue rows без fake resolution. Issue rows не являются retention
  pin; tombstone track UUID и существующий audit остаются, но issue fields в новый
  audit event не копируются.
- Repeat detection обновляет одну active row; recurrence после resolution создаёт
  новый incident. Ни один issue-state transition не создаёт user notification,
  web-inbox record или messenger delivery. Из issue-related данных owner видит только
  текущее derived `has_problems` и generic warning в track views;
  codes/details/history остаются admin-only, а full-lock projection использует
  заданный ниже field allowlist.
- Issue-state transactions сериализуются `SELECT ... FOR UPDATE` на `tracks` row.
  Standalone operation блокирует только track; при необходимости owner lock
  применяется существующий порядок `user -> track`. Advisory locks отсутствуют,
  partial unique active-identity index остаётся DB backstop.
- Commit issue transaction linearize-ит full-lock activation. Canonical single-track
  detail/download держат конфликтующий `FOR SHARE` до response decision/signing,
  mutations — `FOR UPDATE` до commit; active issues повторно проверяются под lock.
  Pre-lock winner завершается до detector commit, post-commit operation видит block.
  Search/map/catalog используют statement snapshot без per-track locks, поэтому
  начатый до commit collection query может вернуть прежний result; in-flight HTTP/S3
  responses не отменяются.
- Preliminary scan только выбирает candidate. Финальный detector-specific check
  выполняется под track lock в той же transaction, которая меняет issue;
  timeout/error откатывает её без изменения state. Под lock выполняется одна attempt
  с hard deadline 10 секунд без retry/backoff; повтор всегда начинает новую
  transaction. Отдельная retry queue отсутствует: CLI/admin возвращают safe error,
  explicit rerun запускается operator/administrator, failed attempt остаётся только
  в logs/metrics.
- Checker возвращает только `problem_present`, `healthy` или `inconclusive`:
  conclusive problem создаёт/обновляет active issue и `last_seen_at`, healthy ставит
  `resolved_at`, inconclusive откатывает transaction без issue mutation.
- S3 exact-key `404`/`NoSuchKey` и completed-read SHA-256/size mismatch являются
  `problem_present`, exact match — `healthy`. Access/permission/`429`,
  DNS/TLS/network/timeout, `5xx`, truncated/unexpected response являются
  `inconclusive` с operational alert и не создают track issue.
- Raw `healthy` требует full streaming GET и пересчёта SHA-256/size по body против
  `raw_objects`, без full-body memory buffer. HEAD только находит missing candidate;
  ETag/custom metadata не являются integrity proof. Hard cap равен stored
  `byte_size + 1`: extra byte даёт conclusive size mismatch и закрывает stream;
  clean short EOF также даёт size mismatch, а transport truncation —
  `inconclusive`. SHA сравнивается только после clean exact-size EOF.
- Issue закрывает только successful detector-specific recheck. Administrator может
  запустить его и добавить append-only audit note, но не force-clear marker; failed
  check оставляет row active, successful заполняет `resolved_at`, history
  сохраняется до physical purge owning track. Отсутствие user notifications не
  создаёт другого resolution path.
- Resolution последней active full-lock issue автоматически пересчитывает effective
  block и меняет только `track_issues.resolved_at`. Lifecycle/moderation state и
  `current_revision_id` не меняются; прежний immutable snapshot снова доступен без
  повторной moderation, только если текущий lifecycle это разрешает. Archive или
  moderator removal и блокировки других issues сохраняются.
- Marker блокирует только reason-scoped capabilities: missing raw запрещает re-parse,
  missing sanitized export — download/publication switch; исправные published
  geometry и metadata view остаются доступны. Effective block — union code mappings,
  не stored scopes; full lock применяется только если хотя бы один active code
  `snapshot_integrity_unknown`. Он блокирует normal content serving,
  owner/content/publication mutations, restore, approve и content-dependent
  moderation. Allowlist содержит minimal status/generic warning, owner archive,
  moderator removal/hide, scheduled purge, admin issue/history read,
  detector-specific recheck/resolution и audit; containment не читает snapshot.
  Owner projection содержит только `track_id`, lifecycle `status`,
  `has_problems = true`, `purge_at` для archived track, localized generic warning и
  доступный archive action. Generic title/short track ID строится без revision read;
  content/revision/export/raw fields, issue semantics и blocked capabilities не
  возвращаются. Archived card показывает purge deadline без restore.
  Blocked owner mutation возвращает terminal `409 Conflict` с
  `Cache-Control: no-store` и stable JSON
  `{"error":"track_temporarily_unavailable"}` либо localized generic HTML. Bot даёт
  neutral acknowledgement, не меняет domain state и обновляет minimal card; rejection
  не retry-ится/DLQ-ится. `423`, `Retry-After` и internal issue data отсутствуют;
  `503` остаётся public/infrastructure response.
  Moderator без admin issue permissions видит full-locked track в existing
  moderation surfaces как snapshot-free placeholder без новой queue: только
  `track_id`, lifecycle `status`, `has_problems = true`, generic warning и
  removal/hide с обязательной причиной. Preview/approve/changes-requested,
  export retry/download и content/revision fields скрыты. Отдельные admin issue
  permissions открывают history/recheck, но не обходят content lock.
  Stale/direct moderator content mutation после auth/permission/visibility checks
  получает тот же terminal `409`/`no-store`/safe JSON error либо generic HTML без
  domain mutation или audit decision; UI возвращается к placeholder. `403`
  означает missing permission, `404` — invisible/removed resource, `503` — infra
  failure. Capability rejection не retry-ится/DLQ-ится и не раскрывает issue.
  Успешный containment removal/hide атомарно создаёт обычный moderation audit,
  locked-on web-inbox record и primary-bot outbox для active owner. Notification
  содержит только removal fact, локализованный public-safe label закрытого
  `reason_code`, short track ID, `TRAILBASE_SUPPORT_URL` и 90-day purge/appeal
  deadline. Начальные codes:
  `policy_violation`, `privacy_or_safety`, `legal_request`, `spam`, `duplicate`,
  `technical_containment`, `other`; code — `text` с DB CHECK, не PostgreSQL enum.
  `technical_containment` показывается как generic «Техническая недоступность».
  Optional audit-only `reason_note` обязателен для `other`, после trim содержит
  1–1000 Unicode code points без control/newline и не показывается owner-у или в
  logs. Full lock, `track_issues.code/detail`, snapshot и internal data не
  копируются. Detector/recheck notification не создают; deactivated-account
  suppression действует без backlog/replay.
  Search/map/catalog results исключают full-locked track. Прямые canonical
  detail/download URLs для otherwise published track отвечают generic `503 Service
  Unavailable` с `Cache-Control: no-store`, без `Retry-After`, metadata или integrity
  details. Private/archived/moderator-removed resources сохраняют одинаковый `404`;
  после resolution следующий request заново проходит publication/capability check.
  Full lock не является non-public lifecycle transition и сам не удаляет/ротирует
  sanitized object: уже выданный presigned URL живёт до исходных пяти минут, cached
  GeoJSON — до 90 секунд, а уже доставленный client content не отзывается. После
  resolution тот же immutable snapshot/export снова обслуживается, если lifecycle
  это разрешает; download proxy не добавляется.
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
  informational flag без automatic moderation decision или изменения
  `submitted_for_review_at`/FIFO position.
- Moderator не редактирует pending revision: author correction после
  `changes_requested` создаёт новый immutable snapshot с повторной validation.
  Decision хранит один `reason_code text NOT NULL` с DB CHECK на
  `metadata_correction`, `geometry_correction`, `privacy_correction`,
  `classification_correction`, `license_or_attribution`, `other`, без enum/array.
  Optional owner-visible note обязателен для `other`, после trim содержит 1–1000
  Unicode code points без control/newline и не логируется; несколько проблем
  перечисляются в note. Removal и issue code namespaces остаются отдельными.
- Первая публикация проходит full-snapshot review. Для изменения published track
  moderator по умолчанию видит field diff и overlay geometry относительно
  `base_revision_id`, с отдельным действием открытия полного pending snapshot.
  Решение применяется только ко всей immutable revision; partial approval
  отсутствует.
- Для resubmit после `changes_requested` default view заменяется на correction diff
  относительно `correction_of_revision_id` с сохранёнными correction code/note;
  total diff относительно `base_revision_id` и полный snapshot остаются вторичными
  views.
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
- Security и action-required moderation notifications (`changes_requested`)
  locked-on; catalog/informational/other moderation delivery configurable.
- `changes_requested` notification содержит localized actionable label закрытого
  correction code и optional owner-visible note из moderation decision; отдельного
  delivery free text нет, note не логируется.
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
  показывает `draft`, `processing`, `pending_review`, `changes_requested` или
  `published` и сначала выводит требующие действия пользователя. Отдельного track
  lifecycle state `rejected` нет.
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
  open/download/new revision/archive для `published`. Delete/archive требуют
  отдельного confirmation.
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
  moderator queue и снова становится eligible после реактивации при
  `export_state = ready`; approve/publish fail-closed проверяет active owner.
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
- Optional web entry: bot выдаёт 10-minute Valkey `web_session` token → safe
  `/auth` GET/POST confirmation → Valkey session cookie. Browser re-auth использует
  тот же flow без отдельного credential/table/cookie/consume endpoint. Token record и
  новая session несут исходный `fresh_authenticated_at`; новое authentication rotate-ит
  current browser session, а ordinary activity/sliding TTL не продлевает absolute
  10-minute freshness.
- Все token-bearing `/auth` GET/POST outcomes, включая confirmation, success,
  invalid/expired и `503`, имеют `Cache-Control: no-store`,
  `Referrer-Policy: no-referrer` и только same-origin static assets. Caddy/app
  access/security logs пишут route без query и form body; raw token/digest отсутствуют
  в errors, analytics и traces.
- Initial token-query `GET /auth` не consume-ит token и всегда redirect-only. Valid
  branch создаёт existing short-lived auth-flow record/cookie+nonce и отвечает `303`
  на clean `/auth`. Malformed/unknown/expired/superseded branch отвечает `303` на
  token-free `/auth` со stable non-secret `result=invalid`; target GET показывает
  generic-invalid без lookup/изменения existing flow, plain `/auth` продолжает current
  confirmation. Marker не добавляет state/cookie. Raw token отсутствует в
  body/redirect/history; JavaScript, `history.replaceState`, client storage и новый
  credential namespace не используются.
- Auth-flow record, cookie и form nonce имеют original `web_session` token absolute
  deadline без fresh interval или sliding от confirmation GET, retry/`503`.
  Expired/missing component показывает generic-invalid page, idempotently очищает
  remaining flow record/cookie и не consume-ит/recover-ит/reissue-ит source token.
- Auth-flow record содержит только source token digest, allowlisted `return_to`,
  original deadline и SHA-256 nonce hash. Independent 128-bit flow-cookie ID адресует
  Valkey record по SHA-256 cookie value; raw source token после redirect не хранится.
  Raw flow ID/nonce остаются только в `HttpOnly` cookie/hidden POST field, bindings
  читаются из token record, raw/digest values отсутствуют в telemetry/DLQ.
- Flow cookie — exact `__Host-trailbase_auth_flow` с `Secure`, `HttpOnly`,
  `SameSite=Lax`, `Path=/`, без `Domain`; `Max-Age`/`Expires` capped remaining source
  deadline. Success, terminal invalid/expiry и explicit cancel очищают её
  `Max-Age=0`; session cookie остаётся отдельной.
- Новый valid initial auth GET с existing flow-cookie атомарно удаляет previous record,
  создаёт replacement и оставляет один active flow на browser. Old tab/form получает
  generic-invalid без side effects; previous source token остаётся unconsumed/unrevoked
  и его link до deadline может снова заменить current flow.
- Failed flow-cookie/form-nonce validation на `POST /auth` не consume-ит source token и
  не удаляет valid flow. Missing/unknown cookie и nonce mismatch дают одну
  generic-invalid response без token/pointer/session mutations. Unknown cookie
  очищается у browser через `Max-Age=0`; valid record при wrong/stale nonce и current
  cookie сохраняются. Общий `/auth` rate limit — 10/min на IP без отдельного
  nonce-attempt counter.
- Non-transient invalid `POST /auth` использует PRG: `303` на token-free `/auth` со
  stable query marker `result=invalid`, затем generic-invalid GET без lookup current
  flow. Cleanup зависит от принятого error class, поэтому wrong/stale nonce для valid
  record сохраняет flow/cookie. Transient PostgreSQL/Valkey failure остаётся retryable
  `503` с `Retry-After` без redirect, terminal marker или credential mutations и
  сохраняет retryable flow/token/cookie.
- После successful PostgreSQL active identity/account check одна atomic Valkey function
  валидирует flow/nonce, source token и active pointer, создаёт либо rotate-ит browser
  session, consume-ит token/pointer и удаляет flow. Все credential mutations имеют одну
  linearization point; distributed PostgreSQL transaction нет. Два concurrent POST
  одной form создают ровно одну session, а проигравший terminal invalid и не revoke-ит
  успешную session.
- Token выпускает только explicit user-initiated «Подтвердить вход» command/callback в
  private one-to-one chat, bound к provider/user/chat/message/requester. При server
  validation/acceptance webhook event injected application UTC clock фиксирует
  `fresh_authenticated_at`; provider-supplied timestamp не влияет на freshness и
  отбрасывается после payload validation. Event dedupe/binding выполняются независимо.
  Recent message, notification/background event или browser activity
  freshness не создают. Второго prompt и PIN/password/TOTP нет.
- `web_session` token/session records не содержат provider event timestamp; auth state
  хранит server `fresh_authenticated_at` и opaque event/identity bindings, необходимые
  для dedupe/consume. В MVP timestamp отсутствует также в operational events, logs и
  metrics; observability ограничена server-time counters по provider/validation result
  без raw payload и отдельной retention policy.
- Единственная fresh-auth metric — counter
  `fresh_auth_confirmation_total{provider,result}` с `provider=telegram|max` и закрытым
  `result=accepted|duplicate|invalid_event|invalid_binding|account_unavailable|internal_error`.
  Identity/event/request/timestamp и дополнительные labels отсутствуют; latency/status
  остаются в общих HTTP/webhook metrics.
- `duplicate` означает exact provider replay только после committed fresh-auth
  acceptance. Replay получает `2xx`, increment-ит лишь `duplicate` и не создаёт новый
  token/link, bot message или session/auth mutation. До commit `internal_error`
  обрабатывается общим worker retry/DLQ и duplicate не является.
- Existing 7-day provider-event dedupe record хранит для fresh-auth только
  `processing|accepted`; отдельный storage отсутствует. Ingress атомарно создаёт
  processing вместе со Stream enqueue. Processing replay получает `2xx` без fresh-auth
  metric/side effects, worker retry/DLQ продолжает original event; после token/link
  issuance worker ставит accepted, replay которого increment-ит duplicate.
- Worker одной atomic Lua/Valkey operation проверяет processing bindings, создаёт
  единственный 128-bit 10-minute `web_session` token record и меняет state на accepted
  до Bot API send. Token/accepted не могут разойтись при crash; external delivery retry
  использует exact same link и не входит в commit boundary.
- Для той же provider identity эта issuance атомарно заменяет per-identity active-token
  pointer, удаляет previous token record и его ещё существующий raw delivery field.
  Previous provider-event record сохраняет accepted marker; отдельные bot message,
  revoke audit и notification не создаются.
- Гонка `/auth` consume old link и new issuance линейризуется atomic Valkey operations
  над pointer. Consume-first удаляет matching token record/pointer и может завершить
  auth; issuance-first заменяет pointer и делает old link terminal invalid. Поздняя
  issuance не отзывает создаваемую/созданную browser session; distributed transaction
  с PostgreSQL не вводится.
- Active-token pointer — отдельный non-secret Valkey key по internal identity UUID,
  содержащий только SHA-256 token digest, с `PEXPIREAT` exact token deadline. Atomic
  consume удаляет его вместе с token record, replacement заменяет; missing/expired
  pointer или target terminal invalid и очищается idempotently. Delivery data, sliding
  TTL и janitor отсутствуют.
- `web_session` token record адресуется только по `SHA-256(raw_token)`, совпадающему с
  pointer digest, без raw token в record/key. 128-bit CSPRNG entropy позволяет
  unsalted SHA-256 без custom HMAC/salt. Raw token остаётся только в link/browser
  request и short-lived delivery field; raw и digest запрещены в telemetry/DLQ.
- Raw link token для crash-safe retry хранится только в short-lived delivery field той
  же accepted dedupe record до successful send или token expiry. Field удаляется после
  delivery/expiry; seven-day marker оставляет только non-secret outcome/bindings. Raw
  secret отсутствует в logs/metrics/traces/DLQ; deterministic derivation, re-issuance и
  второй delivery namespace отсутствуют.
- Valkey 9.x является runtime minimum. Atomic issuance задаёт `HPEXPIREAT` delivery
  field на exact token deadline; successful send выполняет idempotent `HDEL`. Key-level
  accepted marker живёт seven days без polling janitor или отдельного TTL key.
- Если Bot API delivery не завершилась до token/delivery-field expiry, worker
  прекращает её terminally без stale send, re-issuance, изменения accepted marker или
  отдельного user notification. Exact replay остаётся `duplicate`; только новое
  explicit «Подтвердить вход» action создаёт новый provider event/freshness/token/link.
  Failure отражается только общими Bot API delivery metrics/alert без raw token.
- Post-issuance delivery делает максимум пять total attempts с exponential
  backoff/jitter. Retry допускается только для timeout/network, `429` и `5xx`, когда
  следующая attempt с `Retry-After` помещается в исходный token deadline. Остальные
  `4xx`, exhausted budget или retry за deadline terminal сразу: idempotent `HDEL`,
  accepted marker сохраняется, DLQ/late replay отсутствуют. Pre-issuance
  `internal_error` остаётся в общем worker retry/DLQ flow.
- Re-auth разрешён через любую active linked Telegram/Max identity; primary provider
  остаётся delivery preference, не auth trust level. Token bound к exact internal
  identity/provider/user/event; `/auth` consume re-check-ит active membership и
  account status. Unlinked/foreign token terminal invalid, successful freshness
  account-level и не ограничена исходным provider.
- Valid same-user current browser session превращает `/auth` rotation в credential
  refresh той же device continuity: token/CSRF/freshness заменяются, old current
  session revoke-ится, other sessions не меняются и `new_session` notification не
  создаётся. Без valid same-user session successful flow является new login и создаёт
  locked-on security notification.
- Rotated same-browser session сохраняет original `created_at`, получает новый token
  hash/CSRF, bot-derived freshness, current `last_seen_at` и recomputed safe short
  User-Agent; sliding TTL начинается заново на один год. Отдельного
  `reauthenticated_at` нет, ordinary new login получает новый `created_at`.
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
  `404`, otherwise published full-locked track — generic `503` без `Retry-After`,
  metadata или integrity details, а bot никогда не публикует signed URL.
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
- Transparent provider-side SSE/disk encryption raw storage допустимы как deployment
  control: TrailBase не управляет provider keys/key IDs и после GET получает те же
  exact original bytes без application decrypt path.
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
  exact `byte_size`, private SHA-256, lifecycle state и timestamps. Digest
  вычисляется при streaming upload, проверяется при последующих чтениях/re-parse,
  не публикуется и не используется для dedup или API. `upload_jobs` и
  `track_revisions` ссылаются через `raw_object_id`, не копируя storage metadata.
- Mismatch уже `ready` raw завершает parse/re-parse permanent
  `raw_integrity_mismatch` без automatic retry и revision/publication mutation.
  Новый raw state не создаётся; operator alert содержит только `raw_object_id`, а
  row/object остаётся под обычным retention и повторно проверяется при следующем
  чтении. Если восстановление невозможно, пользователь загружает source заново.
- Application/admin repair flow отсутствует. Infrastructure operator может
  восстановить под тем же opaque key только exact bytes с сохранённым SHA-256 из
  MinIO versioning/off-host backup; digest не меняется. Иначе новый upload создаёт
  новый `raw_object_id`, а другой source нельзя подставить в прежнюю raw row.
- После primary purge raw остаётся только в encrypted operator-only off-host backup
  максимум 30 дней, недоступен приложению и не восстанавливается в active service.
  После PostgreSQL/WAL restore и migrations при остановленных web/workers operator
  запускает CLI reconciliation до открытия traffic.
- PostgreSQL является единственным reference source для CLI: любой
  TrailBase-managed S3 key без durable DB reference считается orphan и удаляется со
  всеми versions/delete markers; referenced key не удаляется. Full manifest,
  pending approval state и reconciliation rows не хранятся.
- CLI default — dry-run с orphan counts/total bytes по storage class и nonzero exit
  при DB/S3 scan error. Explicit destructive invocation при остановленных
  web/workers всегда делает новый полный scan и не использует сохранённый dry-run.
- Dry-run не меняет БД. Destructive run удаляет orphans и durable помечает
  referenced-but-missing objects. Полный успешный scan/cleanup с однозначными track
  markers разрешает degraded startup; incomplete scan или unmappable violation
  оставляет traffic закрытым.
- Referenced-but-missing S3 key сохраняет DB reference и ставит связанному track
  durable problem marker. Administrator видит точную безопасную причину, owner —
  только общий факт проблемы без storage/internal details.
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
  `delete_pending` row, пагинированно перечисляет и удаляет по version ID все
  versions/delete markers exact key; prefix matches других keys не удаляются.
  Только empty exact-key listing разрешает physical row delete. Missing
  version/object/row и crash безопасны для replay; wrong state/FK fail-closed
  alert-ятся, key/SHA-256 в outbox не копируются.
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
- Full lock прекращает выдачу новых presigned URLs без object deletion/rotation;
  ранее выданная ссылка остаётся валидной не более исходного five-minute TTL.
- Canonical download разрешает только current published revision. Public
  revision-specific routes отсутствуют; superseded exports не публичны и могут
  очищаться, а новая approved revision переключает тот же stable track URL.
- Supersede/non-public transition ставит high-priority idempotent delete старого
  sanitized object с retry/DLQ/alert. Исключение — private retention последнего
  approved export после moderator removal до `uphold_removal` или 90-day purge.
  Signed URL прекращает работу после delete, а при задержке живёт максимум исходные
  пять минут; download proxy отсутствует.
- Private sanitized export pre-generate-ится для immutable pending revision.
  Approve требует durable `export_state = ready` и атомарно переключает
  published/current pointers; export error оставляет `pending_review` и старый
  current public, без отдельного public `publishing` status.
- До approval object доступен только backend workers. Owner/moderator UI не выдаёт
  presigned URL и не имеет private download route. Download появляется только после
  publication через canonical route.
- Обычная moderator queue содержит только `pending_review` revisions active owners с
  `export_state = ready`. `pending|error`, infrastructure badges и retry action в ней
  отсутствуют; exhausted error обрабатывается в отдельном operator/admin operations
  view с idempotent retry и alert. Owner видит только обычный `pending_review`.
  Snapshot-free full-lock containment остаётся отдельным surface только с removal и
  не зависит от обычной queue eligibility.
- Queue общая: assignment/claim state, lease, heartbeat и hand-off отсутствуют.
  Любой moderator с permission может открыть item; decision повторно проверяет
  eligibility/invariants под row locks. Первый commit сохраняет actual actor в audit,
  конкурентный submit получает deterministic conflict без второго decision, audit
  или notification.
- Eligible items сортируются строго по
  `submitted_for_review_at ASC, track_revisions.id ASC` и листаются keyset-ом по этой
  паре. Priority/escalation, manual bump, SLA buckets и resubmit priority отсутствуют;
  containment и export-error surfaces не входят в FIFO.
- Moderation decision всегда обрабатывает одну revision. Checkboxes/select-all,
  bulk ID payloads/endpoints и batch decision jobs отсутствуют. После успешной
  per-revision transaction UI возвращается в обновлённую общую FIFO queue;
  конкурентный conflict использует тот же refresh.
- Approve использует явную primary submit-кнопку без confirmation modal; final submit
  correction code/note form подтверждает `changes_requested` без второго шага.
  Controls блокируются in-flight, backend re-check остаётся обязательным. Moderator
  removal сохраняет отдельный confirmation из-за hide и retention/appeal lifecycle.
- Ordinary review постоянно показывает approve как primary action и
  `changes_requested` как secondary action с inline correction form. Moderator
  removal вынесен в destructive danger-секцию. Full-lock containment остаётся
  removal-only исключением без approve/correction controls.
- `deferred`, `snoozed` и `needs_second_review` states отсутствуют. Выход из review
  ничего не записывает и оставляет revision в `pending_review` на прежней FIFO
  position, доступной любому moderator-у; отсутствие outcome не является domain
  event.
- Ordinary pending review не имеет `moderation_comments`, drafts/replies, mentions,
  attachments или unread state. Moderator-authored data сохраняются только внутри
  outcome audit: correction note для `changes_requested`, audit-only note для
  removal. Admin `track_issues` notes остаются отдельным operations flow.
- Ordinary moderation queue — одна FIFO list eligible items с keyset controls
  «Назад»/«Далее», без filters, search, saved views/queries или per-filter counts.
  Full-lock containment и export-error operations остаются отдельными surfaces.
- Queue row показывает только track name, localized revision kind и submission
  time/age и умеет только открыть review. Map/geometry/diff/correction/hint previews и
  decision controls доступны лишь на review page; list query не загружает geometry
  или другие тяжёлые snapshot data.
- Moderation queue/review не используют WebSocket, SSE, auto-polling или presence.
  Refresh только request-driven: navigation, manual reload, decision/conflict. Stale
  review разрешён как presentation state, но outcome transaction re-check-ит domain
  state и при race возвращает conflict со свежей queue.
- Проигравшая moderator mutation получает `409 Conflict`, `no-store` и
  `moderation_item_already_decided`; web показывает neutral notice и свежую queue.
  Winner/outcome details и retry отсутствуют, второй audit/notification не создаётся.
- Stale direct HTML GET обработанного ordinary item после permission checks отвечает
  `303` в FIFO queue с neutral one-time notice. Historical ordinary review mode
  отсутствует; полная audit/history находится в отдельном permissioned admin view.
- Approve, `changes_requested` и moderator removal являются append-only immutable
  outcomes: ordinary moderation не позволяет undo, reopen, edit или delete decision.
  После approve исправление создаёт новую owner revision либо отдельный removal,
  после `changes_requested` — новый author resubmit; removal пересматривается только
  отдельным appeal/admin lifecycle. Исходный outcome и reason/note не меняются, а
  корректировка записывается новым audit event.
- Встроенных owner appeal form, `appeals` table, appeal status, attachments/thread и
  отдельной appeal queue нет. Removal notification направляет owner-а по
  `TRAILBASE_SUPPORT_URL` с short track ID и 90-дневным deadline. Support verification
  выполняется out-of-band; результат сохраняет отдельная permissioned admin operation
  новым append-only audit event.
- Appeal management entrypoint — одно exact lookup field по полному track UUID либо
  short track ID из notification/support case. Recent cases, queue, filters, fuzzy
  owner/name search и saved cases отсутствуют. Sensitive context загружается только
  после permission/fresh-auth checks для unique exact match. Collision short ID требует
  полный UUID без automatic selection; unknown/ambiguous lookup возвращает одинаково
  нейтральный результат без context.
- Short track ID вычисляется из последних 12 lowercase hex UUIDv7 `tracks.id` и имеет
  canonical display `trk-xxxx-xxxx-xxxx`. Отдельных column/sequence/generator и UNIQUE
  assumption нет. Input lookup нормализует ASCII case, optional exact `trk-` и group
  hyphens, принимая ровно 12 hex либо canonical full UUID; short path использует
  non-unique expression index и при collision требует full UUID. Первые 8 hex GPX
  filename — отдельный display-only suffix, не lookup ID.
- После lookup management UI показывает одну read-only removal summary, переиспользуя
  historical snapshot renderer вместо второй moderation workspace. Summary содержит
  short/full ID, lifecycle state, original removal time, исходные reason code/note,
  purge deadline, immutable decision snapshot, точный restore target и computed
  eligibility со stable blocking class. Live owner/provider identity, editable fields,
  support text/attachments и full audit timeline не копируются; persisted appeal case
  не создаётся, support verification остаётся out-of-band.
- Case projection содержит один derived/non-persisted `action_state`:
  `decision_ready`, `restore_blocked_full_track_lock`,
  `restore_blocked_export_unavailable`, `window_closed`, `not_current`,
  `already_decided`. Precedence идёт от decided/not-current/window-closed к full lock,
  export unavailable и ready. Ready показывает обе кнопки, restore-blocked state —
  uphold плюс disabled restore с coarse localized reason, terminal states скрывают
  form; decided показывает committed outcome. Issue details отсутствуют. Backend
  recompute-ит state под lock, UI после conflict reload-ит summary и сам eligibility
  не вычисляет.
- Appeal management использует отдельный HTML namespace `/admin/track-appeals`.
  `GET /admin/track-appeals` обслуживает exact `track_ref` lookup/summary; обе outcome
  buttons отправляют `POST /admin/track-appeals/:removal_decision_id/decision` с
  closed outcome/reason/idempotency payload. Mutation требует active admin session,
  fresh auth и CSRF. Htmx имеет full-page fallback; JSON/bot endpoints,
  `/moderation/...` aliases и отдельные uphold/restore routes отсутствуют. Все
  responses — `Cache-Control: no-store`.
- Expired fresh auth между form render и Decision POST останавливает request до
  sensitive reload/mutation и не относится к `422/409/503`. Re-auth flow несёт только
  allowlisted server-side return target на canonical appeal GET с full UUID в
  `track_ref`; submitted outcome/reason/idempotency key не сохраняются в URL,
  auth-flow или session. После re-auth summary и `action_state` вычисляются заново,
  выдаётся новый key, admin повторно заполняет decision; outcome/audit/outbox/
  notification до нового submit отсутствуют.
- После successful re-auth active session получает только generic one-time flash kind
  `appeal_form_discarded`, без identifiers, form values или key. Canonical appeal GET
  атомарно consume-ит его и показывает coarse localized notice о несохранённой
  отправке. Flash не передаётся в query, не входит в `409/503` catalog и исчезает после
  одного render; terminal authoritative summary не показывает form независимо от
  notice.
- Expired fresh auth всегда открывает top-level fresh-auth flow. Full-page POST
  возвращает `303` на server-generated start URL, htmx POST — `200`/`no-store` с
  пустым body и `HX-Redirect` на тот же URL, поскольку htmx не читает response headers
  на `3xx`. Оба используют один server-side `return_to` на canonical appeal GET;
  inline auth fragment отсутствует, invalid session/CSRF остаются отдельными
  fail-closed ветками.
- Appeal re-auth переиспользует bot-issued `web_session` token и `/auth` GET/POST.
  Successful consume rotate-ит current browser session, переносит исходный
  `fresh_authenticated_at` из token record и возвращает по bound `return_to`; отдельной
  re-auth credential state или consume route нет.
- Все token-bearing outcomes этого общего flow используют `no-store`, `no-referrer` и
  only same-origin assets; access/security logs не содержат query/form body, а raw
  token/digest отсутствуют в errors, analytics и traces.
- Initial token-query GET всегда `303` redirect-only без consume: valid branch создаёт
  existing auth-flow record/cookie+nonce. Malformed/unknown/expired/superseded branch
  redirect-ит со stable non-secret `result=invalid`; target GET показывает generic
  invalid без lookup/изменения current flow, plain `/auth` продолжает confirmation.
  Marker не добавляет state/cookie. Raw token отсутствует в body/redirect/history; JS,
  client storage и новый credential state не вводятся.
- Flow record/cookie/form nonce истекают вместе с original token без sliding от GET,
  retry или `503`. Expired/missing component даёт generic-invalid page, idempotently
  очищает remaining flow state/cookie и не consume-ит, не восстанавливает и не
  перевыпускает source token.
- Flow record хранит только source token digest, allowlisted `return_to`, original
  deadline и SHA-256 nonce hash. Independent 128-bit flow-cookie ID использует hashed
  Valkey lookup; raw source token после redirect не хранится. Raw flow ID/nonce
  находятся лишь в `HttpOnly` cookie/hidden POST field, bindings читаются из token
  record, raw/digest values отсутствуют в telemetry/DLQ.
- Общий appeal flow использует `__Host-trailbase_auth_flow`; `Secure`, `HttpOnly`,
  `SameSite=Lax`, `Path=/`, без `Domain`, expiry не позже remaining source deadline.
  Success, terminal invalid/expiry и cancel очищают cookie `Max-Age=0` отдельно от
  session cookie.
- Valid initial GET при existing flow-cookie атомарно заменяет record/cookie, оставляя
  один active browser flow. Old tab/form generic-invalid; previous source token не
  consume/revoke-ится и его link до deadline может снова заменить current flow.
- Failed flow-cookie/form-nonce validation на `POST /auth` не consume-ит token и не
  удаляет valid flow. Missing/unknown cookie и nonce mismatch дают generic-invalid без
  token/pointer/session mutations. Unknown cookie очищается у browser `Max-Age=0`, но
  wrong/stale nonce для valid record сохраняет record и current cookie. Общий `/auth`
  rate limit — 10/min на IP; отдельного nonce-attempt counter нет.
- Non-transient invalid `POST /auth` отвечает `303` на token-free `/auth` со stable
  `result=invalid`; target GET generic-invalid и не lookup-ит current flow. Cleanup
  следует принятому error class, поэтому wrong/stale nonce сохраняет valid flow/cookie.
  Transient PostgreSQL/Valkey failure остаётся retryable `503` с `Retry-After` без
  redirect, terminal marker и credential mutations, сохраняя flow/token/cookie.
- После successful PostgreSQL identity/account check одна atomic Valkey function
  валидирует flow/nonce, source token и active pointer, создаёт либо rotate-ит session,
  consume-ит token/pointer и удаляет flow как одну credential linearization point.
  Distributed PostgreSQL transaction нет; два concurrent POST создают ровно одну
  session, проигравший terminal invalid и успешную session не revoke-ит.
- Appeal link непосредственно выпускает explicit private-chat action «Подтвердить
  вход»; только этот bound validated command/callback создаёт freshness. Recent
  messages, notifications, background/browser activity и второй confirmation не
  используются.
- Действие доступно через любую active linked Telegram/Max identity, не только primary.
  Consume повторно проверяет exact identity/account membership; failure terminal без
  fallback, success создаёт account-level freshness.
- При valid current session того же user consume является same-browser credential
  refresh без `new_session` notification; иначе это ordinary new login с locked-on
  notification. Other sessions re-auth не меняет.
- Same-browser record сохраняет original `created_at`, обновляет
  token/CSRF/freshness/activity metadata и one-year TTL; отдельного
  `reauthenticated_at` нет.
- Actionable form получает server-generated 128-bit CSPRNG base64url
  `idempotency_key` hidden field, общий для обеих buttons и отдельный от CSRF. Outcome
  сохраняет только `idempotency_key_hash` как UNIQUE SHA-256; raw key, отдельная table,
  TTL/cleanup и logging отсутствуют. Same hash с exact normalized
  removal/outcome/reason payload возвращает committed result; mismatch даёт
  `409`/`no-store`/`idempotency_key_reused`, новый key после commit —
  `appeal_already_decided`.
- Successful commit и exact idempotent retry ведут к одному canonical GET summary с
  full track UUID. Full-page POST отвечает `303 See Other`, htmx выполняет client
  navigation к тому же GET без отдельного success fragment. Final summary показывает
  `already_decided`, committed outcome и «Решение сохранено»; refresh не повторяет
  POST. Validation остаётся на form, conflict reload-ит authoritative summary.
- Decision POST error matrix одинакова для htmx/full-page. `422` сохраняет form
  values/same key и показывает field errors; `409` заменяет stale form/key fresh
  summary/action_state со stable code; `503` сохраняет values/key для explicit manual
  retry и не утверждает commit result. Automatic retry/polling нет, `422/409` не имеют
  side effects, uncertain COMMIT разрешается same-key retry. Все responses `no-store`
  и не раскрывают internal errors/details.
- Closed appeal POST catalog: `409` использует только `appeal_already_decided`,
  `appeal_window_closed`, `appeal_not_current`, `appeal_not_restorable` и
  `idempotency_key_reused`; `503` — только `appeal_temporarily_unavailable`.
  `409` mapping к authoritative summary deterministic: decided, window closed, not
  current, recomputed restore-blocked/ready или fresh same-case state. `503` не
  утверждает authoritative state либо commit result. Localized coarse template
  выбирается по status+code без free-form backend/internal details.
- Appeal/admin operation имеет только два terminal outcome: audit-only для lifecycle
  `uphold_removal` и state-changing `restore_after_appeal`. Оба ссылаются на исходный
  removal decision, требуют admin reason и добавляют append-only audit event без
  изменения исходного removal. Generic `resolved`, partial и custom/free-form
  outcomes отсутствуют.
- Одна permission `track_appeal_decide`, mapped только к фиксированной роли `admin`,
  защищает оба outcome; management UI также требует existing fresh auth. Uphold и
  restore не получают отдельных permissions. Ordinary moderator endpoints и bot не
  выполняют operation; checks идут до sensitive audit context, а audit сохраняет
  actual admin actor.
- Final submit заполненной outcome+reason form — единственный confirmation appeal
  decision. Fresh-authenticated management form показывает short track ID, current
  state, выбранный outcome, последствия и обязательную reason. В ней есть две
  визуально разделённые кнопки «Подтвердить удаление» и «Восстановить трек», но нет
  generic «Сохранить», второго modal или отдельного confirm screen. Controls disabled
  in-flight; backend повторно проверяет permission, fresh auth, single-shot и lifecycle
  invariants, retry остаётся idempotent.
- Form показывает static localized hint о 10-minute fresh-auth window, но не exact
  expiry или live countdown. JavaScript timer, polling, auto-refresh/submit отсутствуют;
  backend POST остаётся authoritative и направляет expiry в принятый discard/re-auth/
  fresh-summary flow.
- Actionable GET и Decision POST проверяют один server-side predicate
  `now < fresh_authenticated_at + 10 minutes`; equality означает expired. Скрытого UX
  buffer и второго threshold для render нет, а expiry во время заполнения использует
  тот же discard/re-auth flow.
- Decision POST фиксирует один server `authorization_now` в authoritative mutation
  guard перед preconditions/первой mutation. Если predicate успешен, transaction
  атомарно завершается без повторной wall-clock проверки у commit boundary, даже при
  physical commit после deadline; последующая queue/lock latency не отзывает freshness.
- Appeal GET/POST используют единый injected application-level UTC clock с
  `java.time.Instant` semantics. Client clock и PostgreSQL `now()` не участвуют в
  freshness authorization; app instances имеют синхронизированные system clocks, а
  DB-managed timestamps не становятся вторым freshness authority.
- Этот же application clock создаёт исходный `fresh_authenticated_at` при server-side
  validation/acceptance explicit bot webhook/callback. Provider event timestamp
  отбрасывается после payload validation; dedupe и exact event/identity bindings от
  него не зависят.
- Provider timestamp не входит в `web_session` token/session auth records; они содержат
  только server freshness и opaque bindings для dedupe/consume. Operational events,
  logs и metrics в MVP timestamp также не получают; остаются только server-time
  provider/result counters. Delivery-delay telemetry требует отдельного будущего
  retention contract.
- Appeal re-auth переиспользует только общий
  `fresh_auth_confirmation_total{provider,result}` с закрытыми provider/result values;
  user/chat/event/request IDs, timestamps и другие labels запрещены.
- `duplicate` increment-ится только для exact replay после committed acceptance и
  отвечает `2xx` без новых token/link/message/session side effects; pre-commit
  `internal_error` остаётся retryable через общий worker/DLQ.
- Appeal re-auth использует те же seven-day `processing|accepted` states в existing
  provider-event dedupe record. Atomic ingress claim+enqueue создаёт processing,
  token/link issuance переводит в accepted; отдельной table/key namespace/TTL нет.
- Atomic worker Lua/Valkey commit создаёт один 10-minute token record и accepted state
  до Bot API send; retry внешней доставки переиспользует тот же link без re-issuance.
- При новом confirmation той же provider identity atomic issuance заменяет
  active-token pointer, удаляет previous token record и его ещё существующий raw
  delivery field. Previous event marker остаётся accepted; отдельные bot message,
  revoke audit и notification отсутствуют.
- Previous-link consume и new issuance используют pointer как atomic Valkey
  linearization point. Consume-first может завершить auth после удаления matching
  token/pointer; issuance-first делает previous link terminal invalid. Поздняя issuance
  не отзывает создаваемую/созданную browser session; cross-store transaction нет.
- Pointer хранится отдельным non-secret per-identity Valkey key только с SHA-256 token
  digest и `PEXPIREAT` на original token deadline. Consume удаляет его с token record,
  replacement заменяет; missing/expired pointer или target terminal invalid и
  idempotently очищается. Delivery data, sliding TTL и janitor отсутствуют.
- Token record lookup использует тот же `SHA-256(raw_token)`, что pointer; raw token в
  key/record отсутствует. Unsalted SHA-256 используется для 128-bit CSPRNG token без
  custom HMAC/salt; raw value существует лишь в link/browser request и short-lived
  delivery field, а raw/digest отсутствуют в telemetry/DLQ.
- Exact raw link доступен retry только из short-lived accepted-record delivery field до
  send/expiry; затем остаётся seven-day non-secret marker без secret в telemetry/DLQ.
- Valkey 9.x `HPEXPIREAT` обеспечивает field expiry, successful send — idempotent
  `HDEL`; accepted key TTL не меняется, janitor/отдельный expiry key отсутствуют.
- Expiry до successful Bot API delivery terminally завершает appeal re-auth delivery
  без stale send, re-issuance, изменения accepted marker или отдельного user
  notification. Replay остаётся `duplicate`; новый link требует нового explicit
  «Подтвердить вход» provider event. Failure остаётся только в общих Bot API delivery
  metrics/alert без raw token.
- Post-issuance appeal delivery ограничена пятью total attempts с exponential
  backoff/jitter; retry разрешён только для timeout/network, `429` и `5xx` и только
  внутри token deadline с учётом `Retry-After`. Остальные `4xx`, exhausted budget и
  retry за deadline terminal сразу с idempotent `HDEL`, сохранением marker и без
  DLQ/late replay. Pre-issuance `internal_error` использует общий retry/DLQ flow.
- Appeal outcome переиспользует account-reactivation reason contract:
  `reason_code text NOT NULL` с CHECK на
  `support_request_verified|administrative_correction|other`, без PostgreSQL enum, и
  optional `reason_note`, обязательную для `other`, 1–1000 Unicode code points без
  control/newline. Code/note только в append-only admin audit, не в owner notification
  или logs; validation/UI общие, отдельного catalog нет.
- Appeal outcome single-shot: UNIQUE `removal_decision_id`, first commit wins.
  Same-idempotency-key retry того же outcome возвращает сохранённый результат без
  нового audit/outbox; другой или competing outcome получает
  `409`/`no-store`/`appeal_already_decided` без side effects. Reopen/second appeal нет;
  correction — отдельная permissioned append-only lifecycle operation.
- `restore_after_appeal` повторно публикует прежний immutable approved
  `current_revision_id`, если он был до removal, без новой ordinary moderation. Без
  прежней approved revision track становится только private/editable и требует нового
  owner resubmit. Removed pending revision остаётся decided/discarded и никогда не
  возвращается в queue.
- После moderator removal private sanitized export последнего approved current
  snapshot хранится до `uphold_removal` либо 90-day purge. Canonical routes отвечают
  `404`, новые presigned URL не выдаются, object доступен только backend. Restore одной
  transaction возвращает snapshot в published и пишет audit/outbox без regeneration
  или `restoring`; uphold/purge удаляют object. Unapproved pending export немедленно
  discard-ится и удаляется.
- Uphold сохраняет original `purge_at`: sanitized export удаляется сразу, остальные
  retained data — в исходный 90-day deadline. Restore атомарно ставит
  `purge_at = NULL`; extension/new timer отсутствуют, error/conflict/retry срок не
  меняют. Outcome и purge worker сериализуются track lock: purge-first делает restore
  invalid, restore-first заставляет worker no-op после re-check.
- Первый новый appeal outcome под track lock требует current active removal,
  отсутствующий outcome и `now() < purge_at`. Expiration закрывает uphold и restore
  даже при lag purge worker-а; support request окно не продлевает, newer lifecycle
  state закрывает обе операции. До deadline full-track lock и unavailable retained
  export блокируют только restore, audit-only uphold остаётся доступным. Committed
  outcome по-прежнему обслуживает same-outcome idempotent retry.
- Restore под track lock требует current `moderator_removed`, matching active
  `removal_decision_id`, unexpired purge deadline, отсутствие newer lifecycle event и
  full-track lock, плюс ready retained export для published branch. Иначе
  `409`/`no-store`/`appeal_not_restorable` не создаёт outcome, audit/outbox или delete;
  temporary block допускает retry до deadline. Full lock не блокирует audit-only
  uphold.
- Первый committed appeal outcome атомарно создаёт locked-on owner web-inbox record и
  primary-bot outbox. Uphold содержит только confirmation fact, short track ID и purge
  deadline; restore — canonical link для published либо edit/resubmit для private
  branch. Admin reason/note, actor, audit ID/context скрыты. Error/conflict/idempotent
  retry notification не создают; disabled owner следует moderation suppression без
  backlog/replay.
- Changes-request или moderator removal pending revision атомарно переводит private
  export в `discarded` и ставит object delete. Новый revision после changes-request
  создаёт свой export; late worker completion не оживляет discarded state и чистит
  созданный object через retry/DLQ/alert.
- Обычная track moderation имеет только approve/publish и `changes_requested`;
  terminal policy, privacy/safety, legal, spam и duplicate cases используют
  moderator removal. `rejected` остаётся допустимым только в lifecycle других
  independently moderated entities, например POI links и tag requests.
- First-publication review показывает полный snapshot; update review по умолчанию
  показывает field diff и overlay новой geometry с сохранённым `base_revision_id`.
  Полный pending snapshot доступен отдельным действием, но любое решение относится ко
  всей revision. Approval stale baseline не выполняется и не делает automatic rebase.
- Author resubmit после `changes_requested` хранит same-track
  `correction_of_revision_id` возвращённой immutable revision и по умолчанию
  показывает correction diff с её сохранёнными code/note. Total diff от
  `base_revision_id` и полный snapshot доступны как вторичные views.

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
