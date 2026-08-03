# Контракт: GPX intake, geometry и object storage

Часть [Implementation Contract](../../IMPLEMENTATION-CONTRACT.md). При противоречии с другими документами действует этот контракт.

## 10. GPX intake и validation

- Web и bot uploads используют один pipeline. Любой GPX attachment в private
  one-to-one chat от identity, прошедшей provider webhook validation, автоматически
  начинает новый upload flow после проверки account quota и лимита трёх active
  flows; browser session и предварительная команда не требуются. `/upload` только
  предлагает прислать файл и не создаёт обязательный pending mode. Attachment другого
  типа не создаёт job и получает короткую инструкцию о допустимом GPX-формате.
  Group/channel upload не создаёт job или draft и предлагает продолжить в private
  chat.
- Полный upload flow может завершаться в Telegram/Max chat: приём файла, validation,
  выбор обязательных metadata, preview статуса parse и отправка revision на moderation.
  Web deep-link остаётся дополнительным, а не обязательным шагом.
- Один пользователь может вести в private chat несколько upload flows одновременно в
  пределах лимита трёх active upload flows. Неявного «текущего upload» нет:
  status, controls и metadata prompts всегда связаны с конкретным `upload_job_id`.
  Свободный текст принимается как metadata только в reply на prompt этого job; иначе
  bot просит явно выбрать upload.
- После успешного parse bot использует редактируемую draft summary card, а не
  обязательный последовательный wizard. Name и Activity явно отмечены как required;
  actions открывают bound prompts для этих и optional metadata, а пользователь
  заполняет поля в любом порядке. «Отправить на модерацию» при незаполненных required
  полях не выполняет mutation и перечисляет недостающие поля. Готовый draft сначала
  показывает final summary с CC BY 4.0 reminder/confirmation.
- После принятия attachment bot сразу отправляет одну status card, привязанную к
  `upload_job_id`, и best-effort редактирует её при переходах
  download/validation/parse вместо отдельных сообщений на каждый этап. Terminal
  success превращает card в draft summary с дальнейшими actions, terminal failure —
  в безопасное объяснение с retry action. Если provider edit недоступен или
  завершается ошибкой, bot отправляет одну replacement card; durable domain job не
  запускается повторно и не зависит от состояния сообщения.
- Лимит raw GPX — 10 MiB; лимит всего multipart request — 12 MiB. Provider-reported
  size/`Content-Length` проверяется, но download дополнительно имеет streaming hard cap.
- Принимается только несжатый GPX/XML 1.0 или 1.1. `.gz` и ZIP в MVP запрещены.
- Проверяется структура, а не строгая XSD. Неизвестные extensions безопасно
  игнорируются.
- DTD, XInclude, external entities и parser network access запрещены.
- Максимум 250 000 GPX points и XML depth 64.
- `NaN`, infinity и координаты вне диапазона отклоняют файл. Сегмент короче двух точек
  игнорируется с warning; отсутствие валидных сегментов завершает job ошибкой.
- Каждый `<trkseg>` становится отдельным сегментом 2D PostGIS `MultiLineString`.
  Разрывы не соединяются искусственной линией.
- Один принятый attachment создаёт ровно один TrailBase draft, даже если GPX содержит
  несколько `<trk>`; автоматического split на несколько drafts нет. Валидные
  `<trkseg>` всех `<trk>` становятся компонентами canonical `MultiLineString` в
  порядке документа.
- Отдельный segment manifest в MVP не хранится: границы и deterministic order
  сегментов уже задаются компонентами `MultiLineString`, а исходная иерархия
  `<trk>/<trkseg>` остаётся в original raw GPX и доступна только через re-parse.
- Parse result сохраняет только scalar diagnostics `source_track_count`,
  `source_route_count` и `valid_segment_count`, без source indexes или metadata
  отдельных `<trk>`/`<rte>`. При нескольких source elements выбранного geometry kind
  draft summary и final summary показывают неблокирующий warning с source/valid
  counts рядом с обычным map preview. Отдельного confirmation gate нет: final submit
  подтверждает весь snapshot; пользователь может отменить draft и загрузить
  отдельные файлы, если нужны отдельные catalog tracks.
- Route-only GPX поддерживается: каждый `<rte>` становится сегментом. Если есть и
  `<trk>`, и `<rte>`, используются tracks, а routes показываются warning. Один
  route-only attachment с несколькими `<rte>` также создаёт один draft без automatic
  split.
- Waypoints показываются в preview и могут стать POI suggestions, но не создают
  locations автоматически.
- GPX `<name>` и `<desc>` используются только как неподтверждённые defaults формы.
  Initial Name выбирается в порядке: непустой file-level `<metadata><name>`; затем
  единственное distinct непустое `<trk><name>` среди всех tracks (одинаковые значения
  считаются одним); затем безопасно нормализованный stem provider filename без path,
  control characters и расширения. При нескольких разных `<trk><name>` parser не
  выбирает первый и не объединяет строки, а показывает warning и переходит к
  filename fallback. После копирования это обычное редактируемое draft name, а raw
  filename отдельно не хранится и публично не показывается. Если все источники
  отсутствуют, Name остаётся пустым. Name общий, description — plain-text map по
  языкам. Для route-only input применяется тот же порядок, но element-level
  candidate берётся из единственного distinct непустого `<rte><name>`; несколько
  разных route names дают warning и filename fallback.
- Initial Description выбирается из непустого file-level `<metadata><desc>`, затем из
  единственного distinct непустого `<trk><desc>` среди всех tracks; одинаковые
  значения считаются одним. При нескольких разных `<trk><desc>` parser ничего не
  выбирает и не объединяет, показывает warning и оставляет optional Description
  пустым для ручного ввода. Выбранный default остаётся неподтверждённым и записывается
  только в owner UI locale на момент upload. Для route-only input применяется тот же
  порядок с единственным distinct непустым `<rte><desc>`; несколько разных route
  descriptions дают warning и пустой optional field.
- Caption исходного GPX attachment не копируется в draft Description или иные domain
  metadata: он может содержать контекст чата или инструкцию. Description получает
  только выбранный по этому precedence GPX `<desc>` default либо текст, явно
  введённый через bound Description prompt. Caption остаётся лишь внутри принятого
  24-hour retention raw webhook payload и не становится domain content.
- Если GPX `<desc>` не содержит locale, он записывается как неподтверждённый default
  в ветвь текущего сохранённого UI locale owner на момент upload (`ru` или `en`).
  Language detection, translation и копирование в обе ветви не выполняются. Owner
  может изменить language binding и добавить другую версию через metadata actions;
  последующая смена UI locale не переносит уже сохранённый Description.
- При отображении Description presentation layer сначала использует requested UI
  locale, затем другую из `ru`/`en` с явной меткой фактического языка. Автоматического
  перевода нет; если обе ветви пусты, секция отсутствует. JSON API возвращает полный
  description map с обеими language branches и не применяет presentation fallback.
- Перед submit на moderation обязательны непустой final name и явно выбранный primary
  activity. Автоматическое agreement уже записано при `/start` и не является
  отдельным submit gate. Неизменённый GPX name считается подтверждённым только вместе
  с final submit summary; classifier suggestion не считается выбором activity.
  Summary напоминает о CC BY 4.0 и содержит license link.
- Description optional. Difficulty, season, tags и auto-derived metrics не блокируют
  submit. Metrics показываются в final summary и подтверждаются вместе со всем
  snapshot без отдельного подтверждения каждого поля.
- Limits: name и provider display name — 200 Unicode code points, description —
  10 000 на язык, provider username — 100. Импортированное превышение обрезается с
  warning; пользовательский ввод сверх лимита отклоняется.

## 11. Geometry, elevation и duration

- Canonical spatial geometry всегда 2D `MultiLineString` SRID 4326.
- Canonical length — `ST_Length(geometry::geography)`. Parser length используется
  только для preview.
- Simplified geometries вычисляются в EPSG:3857 через
  `ST_SimplifyPreserveTopology` и возвращаются в 4326:
  - `geometry_simplified_z11`: 40 м;
  - `geometry_simplified_z13`: 10 м;
  - `geometry_simplified_z15`: 2 м.
- Elevation samples не помещаются полностью в PostgreSQL. Revision содержит
  LTTB-профиль максимум 2 000 samples: distance, elevation, lat/lon, segment index.
- При покрытии elevation менее 90% точек профиль и gain/loss считаются недоступными;
  отсутствующие значения не превращаются в нули.
- Gain/loss: ресемплирование по дистанции 10 м, median filter 5 samples, порог
  изменения 3 м. Revision хранит версию алгоритма.
- Сохраняются `elapsed_duration_s` и `moving_duration_s`. Default canonical
  `duration_s` выбирается из `moving_duration_s`, затем из `elapsed_duration_s`, иначе
  остаётся unknown. Оба рассчитанных значения, canonical value и source показываются
  в final submit summary без отдельного обязательного подтверждения duration.
- Пользователь может заменить canonical duration вручную или выбрать unknown; это
  меняет `duration_s`/`duration_source`, но не удаляет auto-derived moving/elapsed
  values из revision snapshot.
- Manual duration хранится как целое число секунд в диапазоне
  `1..31_536_000` (365 дней). Значение вне диапазона отклоняется. Unknown хранится как
  `duration_s = NULL`, `duration_source = unknown`; ноль не является duration.
- Для outlier comparison используется тот же default auto candidate:
  `moving_duration_s`, если доступен, иначе `elapsed_duration_s`. Warning требует
  отдельного подтверждения, когда
  `max(manual / auto, auto / manual) > 10`, но не запрещает сохранить значение.
  Ровно 10× warning не вызывает; без auto candidate comparison не выполняется.
- Outlier acknowledgement хранится durable в PostgreSQL и привязано к exact
  `manual_duration_s`, comparison source/value и версии duration algorithm.
  Confirmation mutation проверяет текущую lease generation и атомарно сохраняет
  acknowledgement. Оно переживает resume/takeover, но инвалидируется при изменении
  manual value, reparse или версии алгоритма. Valkey хранит только одноразовый
  prompt/control token и не является источником acknowledgement state.
- Moderation summary для acknowledged outlier показывает canonical manual duration,
  comparison auto value/source, ratio и факт acknowledgement. Это informational flag:
  он не вызывает automatic moderation decision, не меняет
  `submitted_for_review_at` или FIFO position.
- Moderator не изменяет duration или другие поля pending revision напрямую: он
  approve-ит snapshot как есть либо возвращает `changes_requested` с обязательной
  причиной. Исправление делает автор в новом immutable full revision без обязательной
  повторной загрузки GPX; validation и outlier acknowledgement вычисляются заново.
  Отдельная audited administrative repair operation не служит shortcut moderation.
- Moving time включает интервалы со скоростью не ниже 0,5 км/ч. Интервалы с
  неположительным временем, gap больше 10 минут или физически невозможной скоростью
  исключаются с warning. Алгоритм версионируется.
- `duration_source`: `gpx_moving`, `gpx_elapsed`, `manual`, `estimated`, `unknown`.

## 12. Object storage и upload jobs

- Первый production использует MinIO в Compose, но приложение зависит только от
  configurable S3 endpoint/bucket/credentials.
- Все buckets private. Object keys — opaque UUID с техническим prefix (`raw/`,
  `exports/`, `photos/`), без user/track IDs и исходных filenames.
- Private raw GPX не шифруется и не преобразуется приложением: S3 object содержит
  exact original upload bytes. Application crypto envelope, data/master keys,
  keyring, rewrap и decrypt path для raw отсутствуют.
- Transparent provider-side SSE или disk encryption raw storage допустимы как
  deployment control. TrailBase не управляет этими ключами, не хранит key ID и
  получает после S3 GET те же exact original bytes; application decrypt path не
  появляется.
- Raw GPX служит только внутренним source для parse/re-parse. Public или owner-only
  download route, presigned URL и HTTP streaming endpoint отсутствуют; raw читают
  только backend workers. Пользователь может скачать только опубликованный sanitized
  GPX.
- После успешного parse raw не имеет отдельного TTL и хранится, пока
  связанный draft/track не очищен физически. Он сохраняется во всех lifecycle states,
  включая published и archive/appeal retention, и всё время учитывается в user
  raw-storage quota. Raw удаляется при подтверждённом физическом удалении draft либо
  purge track; 24-hour janitor применяется только к incomplete jobs и orphan objects.
- Sanitized GPX не шифруется приложением: S3 хранит exact export bytes в private
  bucket, а клиент получает их напрямую по presigned HTTPS URL. Transparent
  provider-side SSE или disk encryption допустимы как deployment control, но не
  меняют object bytes или TrailBase API; backend decrypt proxy отсутствует.
- Каждый новый upload всегда создаёт новый immutable exact raw object;
  content-based dedup отсутствует даже внутри одного user, а также cross-track и
  cross-user. Re-parse или metadata-only revision внутри того же track lineage без
  нового upload ссылается на существующий raw object и не копирует source bytes.
  Raw-storage quota считает физический object один раз; cleanup удаляет его только
  после исчезновения последней durable reference. Private SHA-256 вычисляется при
  streaming upload, сохраняется для integrity checks при последующих чтениях/re-parse
  и не публикуется, не используется для dedup или различимого поведения API.
- SHA-256 mismatch при чтении уже `ready` raw обрабатывается fail-closed без нового
  raw lifecycle state. Текущий parse/re-parse завершается permanent
  `raw_integrity_mismatch` без automatic retry, не создаёт и не меняет revision или
  publication state и отправляет operator alert только с `raw_object_id`. Raw
  row/object остаётся под обычным retention для диагностики или восстановления;
  каждая следующая попытка снова проверяет digest. Если восстановление невозможно,
  пользователь получает нейтральную просьбу загрузить исходный файл заново.
- Application/admin repair flow для raw в MVP отсутствует. Operator может через
  infrastructure procedure восстановить под тем же opaque key только exact bytes,
  совпадающие с сохранённым SHA-256; stored digest не изменяется. Если совпадающей
  копии в MinIO versioning/off-host backup нет, пользователь делает новый upload с
  новым `raw_object_id`. Другой source нельзя записать под старый key, привязать к
  прежней raw row или выдать за исходный object.
- После primary purge raw может оставаться только в зашифрованном off-host backup не
  более 30 дней. Backup доступен только operator-ам и никогда не монтируется
  приложением; purged raw нельзя восстанавливать в active service.
- Referenced S3 key, отсутствующий в storage, не приводит к удалению DB reference.
  Integrity detector связывает нарушение с затронутым track и
  устанавливает его durable problem marker с admin-only reason code и безопасным
  техническим пояснением. Тот же marker используется для обнаруженных проблем других
  track parameters. Administrator видит точную причину; owner во всех интерфейсах
  видит только нейтральный факт проблемы с записью, без storage/internal details.
- Problem reason блокирует только capabilities, зависящие от неисправной части.
  Missing raw запрещает re-parse, но не скрывает исправную published geometry;
  missing sanitized export запрещает download и publication switch, но не metadata
  view. Generic owner warning показывается при любой причине. Полный track lock
  применяется только когда reason означает, что целостность всего snapshot нельзя
  подтвердить. При нём fail-closed блокируются normal content serving,
  owner/content и publication mutations, restore, approve и другие
  content-dependent moderation decisions. Allowlist содержит только minimal status
  projection с generic warning, owner archive и moderator removal/hide как
  containment, scheduled purge, admin issue/history read, detector-specific
  recheck/resolution и append-only audit; containment не читает и не публикует
  snapshot.
- Active storage issue становится resolved только после detector-specific successful
  recheck фактического object и ожидаемых integrity/metadata invariants. Administrator
  может запустить recheck и добавить append-only audit note, но не установить
  `resolved_at` вручную. Failed recheck оставляет issue active. Row не удаляется:
  успешный checker заполняет `resolved_at`, сохраняя историю.
- PostgreSQL table `raw_objects` является единственной durable ownership/storage
  boundary для raw: row хранит `owner_id`, opaque S3 key, exact `byte_size`, private
  SHA-256, lifecycle state и timestamps. `upload_jobs.raw_object_id` и
  `track_revisions.raw_object_id` ссылаются на эту row; revisions не копируют object
  key или storage metadata. Quota считает distinct `raw_objects`, а cleanup
  определяет последнюю durable reference по БД.
- Retained `track_revisions.raw_object_id` всегда pin-ит raw; FK обязательный для
  GPX-derived revision и использует `ON DELETE RESTRICT`. Nullable
  `upload_jobs.raw_object_id` использует `ON DELETE SET NULL` и является pin только
  пока job может продолжить upload/parse либо explicit transient retry до 24-hour
  incomplete deadline. После successful revision commit job больше не pin-ит object,
  потому что его удерживает revision; cancel/permanent failure снимает job pin и
  сразу разрешает cleanup. Историческая job row может сохранить correlation до
  physical raw delete, после чего FK становится `NULL`.
- Добавление новой revision reference и last-reference cleanup сериализуются одним
  порядком: owner `users` row, затем `raw_objects` row `FOR UPDATE`. Reuse после lock
  проверяет совпадение owner, принадлежность existing raw тому же track lineage и
  `state = ready`, затем вставляет revision reference. Cleanup под теми же locks
  заново проверяет pin predicate и только затем атомарно ставит `delete_pending` с
  outbox command. `delete_pending` необратим: если cleanup победил, поздний reuse
  отклоняется и требует нового upload. `ON DELETE RESTRICT` остаётся DB-защитой от
  physical row delete при retained revision.
- `raw_objects.state` имеет только `pending`, `ready`, `delete_pending`. Row
  `pending` создаётся до S3 PUT; только checksum-validated `ready` можно привязать к
  revision. Потеря последней durable reference атомарно меняет `ready` на
  `delete_pending` и пишет idempotent cleanup outbox command. Незавершённый
  `pending` по 24-hour janitor также переходит в `delete_pending`. После успешного
  idempotent S3 delete worker физически удаляет row. Upload failure хранит
  `upload_jobs`, delete retries/DLQ — cleanup command; отдельных `error` и `deleted`
  states у `raw_objects` нет.
- Raw cleanup outbox command содержит только `raw_object_id`, без S3 key или SHA-256.
  Worker по ID читает current `delete_pending` row и opaque key, пагинированно
  перечисляет versions/delete markers этого exact key и удаляет каждый по version ID.
  Prefix matches других keys игнорируются; listing/delete повторяются, пока exact key
  полностью не отсутствует. Empty listing, missing version и уже удалённый object
  считаются success; только после подтверждённого отсутствия всех versions/markers
  worker физически удаляет row. Если row уже отсутствует, replay считается success.
  State, отличный от `delete_pending`, либо retained revision FK обрабатываются
  fail-closed через retry/DLQ и operator alert. Crash между S3 operations и DB delete
  безопасен: replay повторяет exact-key enumeration и завершает cleanup.
- Для user raw-storage quota каждый новый `pending` атомарно резервирует полный
  per-file limit 10 MiB, не доверяя provider size или `Content-Length`. После
  checksum-validated upload переход в `ready` заменяет reservation фактическим
  byte size. `delete_pending` не входит в quota с момента commit:
  terminal upload failure переводит row туда немедленно, а janitor остаётся
  fallback. Задержка/DLQ физического S3 delete считается operational storage leak и
  не блокирует новые uploads owner-а.
- Quota не кэшируется в `users`. Каждая quota-changing transaction сначала блокирует
  owner row `FOR UPDATE`, затем вычисляет по `raw_objects` indexed SQL sum:
  `pending` даёт 10 MiB, `ready` — `byte_size`, `delete_pending` — zero.
  Новый `pending` вставляется только после проверки суммы вместе с reservation в той
  же transaction. Partial covering index по `owner_id`/`state` для `pending|ready`
  включает `byte_size`; денормализованный counter и reconciliation job
  отсутствуют.
- S3 и PostgreSQL согласуются через `upload_jobs`: pending DB row, object write,
  validation/parse, transactional revision creation, terminal status.
- Incomplete jobs и orphan objects старше 24 часов удаляет janitor.
- HTTP upload сохраняет exact original bytes в private S3 object и ставит parse job.
  Web UI опрашивает status htmx-запросом каждые две секунды до terminal state. Для
  bot upload worker
  best-effort обновляет одну job status card; terminal success показывает draft
  summary и продолжает metadata/moderation flow, terminal failure — безопасную
  причину и retry action. Provider edit failure создаёт replacement card, но не
  повторяет domain job.
- Transient S3/Valkey/PostgreSQL failures повторяются до трёх раз. Invalid GPX,
  отсутствующая geometry и превышение limits завершаются без retry.
- После исчерпания автоматических попыток transient infrastructure failure показывает
  explicit retry. Он повторно проверяет active account/quota, атомарно занимает
  свободный upload slot и ставит тот же `upload_job_id` в очередь без второго job;
  сохранённый raw object используется повторно. Если slot недоступен, job не
  меняется и bot показывает active flows. Повторный callback идемпотентен и не
  запускает две попытки одновременно.
- Invalid GPX, отсутствие geometry и превышение limits не повторяют те же bytes:
  status card предлагает отправить исправленный GPX как новый attachment. Если
  raw object transient-failed job не был сохранён или уже очищен, retry также
  просит прислать файл заново вместо создания попытки без source.
- После успешного parse создаётся постоянный private track draft. Он не истекает и
  доступен только owner и moderators; unlisted/secret-link доступа нет.
- Если owner деактивирован до revision commit, уже начатый worker может завершить job
  только созданием private draft и terminal technical status. Он не отправляет draft
  на moderation и не создаёт publication side effects; orphan cleanup остаётся
  обычной ответственностью janitor.
- Явный cancel после успешного parse закрывает active chat upload flow и освобождает
  slot, но не удаляет private draft или raw. Draft остаётся под quota и может
  быть продолжен через `/drafts` или web. Физическое удаление draft/raw — отдельное
  явное действие с подтверждением.
- User quota: 100 private drafts и 1 GiB raw storage, с admin override. Проверка raw
  quota учитывает fixed 10 MiB reservation для `pending`, actual byte size для
  `ready` и zero для `delete_pending`.
- Максимум три active upload flows на пользователя. Slot занят на стадиях
  intake/download/parse и после успешного parse при ожидании обязательных metadata или
  submit на moderation. Slot освобождается только после submit, явной отмены или
  terminal intake/parse failure.
- Resume сохранённого draft через `/drafts` атомарно занимает active slot. Если все три
  slots заняты, draft остаётся закрытым, а bot показывает active flows и предлагает
  завершить или отменить один.
- Один draft имеет максимум один global active edit/upload lease во всех interfaces:
  Telegram, Max и web. Повторный вход показывает существующий flow и предлагает явный
  takeover. Takeover атомарно переносит lease без нового slot, меняет lease generation
  и инвалидирует старые prompts/controls; каждая metadata mutation проверяет текущую
  generation, чтобы предотвратить lost updates.
- Takeover требует обычной owner authentication и explicit confirmation, но не fresh
  bot authentication: web использует session-bound CSRF, chat — validated private
  webhook identity.
- После commit takeover прежний bot prompt best-effort редактируется: controls
  удаляются, показывается нейтральное сообщение о продолжении в другом interface.
  Ошибка provider edit не rollback-ит lease и только логируется/метрится.
- Для web отдельный realtime transport не добавляется. Следующая mutation со stale
  generation получает `409` с сообщением о takeover и reload/resume action.
- Post-parse lease закрывается после 24 часов без успешного user metadata/control
  action. Успешная mutation продлевает idle deadline в той же транзакции; status reads,
  polling и worker events его не продлевают. Expiry освобождает slot, сохраняя
  draft/raw. Intake/download/parse используют отдельный 24-часовой incomplete-job
  timeout.
- Global draft lease, generation, active interface, last user action и three-slot
  accounting хранятся в PostgreSQL рядом с durable upload/draft state. Acquire,
  takeover, generation validation, metadata mutation и slot transition выполняются
  транзакционно. Valkey хранит только ephemeral prompt/control tokens; его потеря не
  меняет ownership, lease или slot count.
- Durable flow row содержит `slot_no` с `CHECK (slot_no BETWEEN 1 AND 3)` и
  `closed_at`. Partial unique indexes для `closed_at IS NULL` гарантируют уникальность
  `(user_id, slot_no)` и, при ненулевом draft, `draft_id`. Acquire транзакционно
  выбирает свободный slot и обрабатывает unique conflict; correctness не зависит от
  race-prone проверки `COUNT(*) < 3`.
- Перед каждым upload/resume acquire та же транзакция синхронно закрывает eligible
  expired post-parse leases пользователя, увеличивает их generation и освобождает
  slots, затем выделяет `slot_no`. Janitor вызывает ту же domain operation как
  фоновый fallback; корректность acquire не зависит от расписания janitor.
- Slot lifecycle operations `upload/resume/cancel/submit/expiry` сначала выполняют
  `SELECT ... FOR UPDATE` durable `users` row и соблюдают единый lock order
  `user -> flow -> draft`. Операции разных users не блокируют друг друга; partial
  unique indexes остаются DB backstop. PostgreSQL advisory locks не используются.
- Cancel до создания draft транзакционно ставит `cancel_requested_at` и увеличивает
  flow generation, но slot остаётся занят до terminal `cancelled`. Worker проверяет
  cancel перед S3 write, parse и revision commit.
- Если cancel выигрывает race, draft/revision не создаётся, temporary raw object
  удаляется best-effort и остаётся под janitor fallback. Если revision commit уже
  победил, применяется post-parse cancel: draft/raw сохраняются, flow закрывается и
  slot освобождается.
- Новые uploads получают `503 Retry-After`, если parse queue больше 1 000 jobs или
  старейшая job ждёт больше 10 минут. Catalog и auth остаются ready.
- `parse-worker` использует отдельный stream и стартовую concurrency 2 на container.
  `bot-worker` использует другой stream, чтобы uploads не задерживали auth.
