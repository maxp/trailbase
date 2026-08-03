# Контракт: track integrity и revisions

Часть [Implementation Contract](../../IMPLEMENTATION-CONTRACT.md). При противоречии с другими документами действует этот контракт.

## 13. Track aggregate и revisions

- `tracks` хранит stable identity: ID, неизменяемый `owner_id`, lifecycle timestamps,
  status и `current_revision_id`.
- Track имеет durable problem marker для storage и других data-integrity нарушений.
  Вместе с marker сохраняются admin-only reason code и безопасное пояснение без
  credentials, raw GPX или чувствительного payload. Administrator видит точную
  причину в management UI; owner в web и private Telegram/Max chat видит только
  нейтральное сообщение, что с записью трека обнаружена проблема.
- PostgreSQL `track_issues` хранит несколько одновременных issues: `id`, `track_id`,
  `code text NOT NULL`, `subject_type text NOT NULL`, `subject_id UUID NOT NULL`,
  admin-only `detail text NOT NULL`, `detected_at`, `last_seen_at` и nullable
  `resolved_at`. DB CHECK требует
  `char_length(detail) BETWEEN 1 AND 1000`. Detail формируется только из
  code-specific безопасных шаблонов; raw exceptions, HTTP headers/provider body,
  GPX content и credentials не сохраняются. Машинная семантика остаётся в `code`;
  `jsonb` для detail не используется.
  Шаблоны образуют закрытый application-каталог по `code`, и каждый имеет явный
  whitelist подстановок. В MVP разрешены только тип mismatch и expected/observed
  byte count там, где count известен точно. Если oversized read остановлен на
  `expected + 1`, detail сообщает лишь нижнюю границу observed size, а не точный
  размер. `subject_id`, object key/bucket, filename, digest, provider
  response/status, exception и operator free text не подставляются; ручные
  пояснения администратора остаются только append-only audit notes.
  Rendered detail хранится только на canonical English, независимо от `ru`/`en` UI
  locale, и не переписывается при её смене. Admin UI локализует labels и actions,
  но показывает stored detail как canonical diagnostic text.
  `code` ограничен DB CHECK значениями `raw_object_missing`,
  `raw_integrity_mismatch`, `sanitized_export_missing`,
  `snapshot_integrity_unknown`; PostgreSQL enum не используется. Отдельной
  capability `scope` column нет: blocked capabilities детерминированно выводятся из
  закрытого code mapping:
  - `raw_object_missing`, `raw_integrity_mismatch` → `reparse`;
  - `sanitized_export_missing` → `download`, `publication_switch`;
  - `snapshot_integrity_unknown` → `full_track_lock`.
  Unknown code не принимается; новый code требует migration существующего CHECK,
  checker, subject-type constraint и capability mapping. `subject_type` ограничен
  DB CHECK значениями `track`, `raw_object`, `revision`; PostgreSQL enum и
  неизвестные значения не используются.
  Отдельный compound DB CHECK допускает только пары:
  `raw_object_missing|raw_integrity_mismatch` с `raw_object` и
  `sanitized_export_missing|snapshot_integrity_unknown` с `revision`. Начальный
  набор codes не допускает row с `subject_type = 'track'`; первый track-wide code
  одной migration расширяет CHECK codes и этот pair CHECK.
  Track-wide issue использует `track`/`track_id`; raw —
  `raw_object`/`raw_objects.id`; revision и её sanitized export —
  `revision`/`track_revisions.id`. Partial unique constraint по
  `(track_id, code, subject_type, subject_id)` для `resolved_at IS NULL` допускает
  только один active incident этой identity; nullable subject/sentinel и
  `NULLS NOT DISTINCT` не используются.
  `track_id` является обычным FK, а polymorphic `subject_id` намеренно не имеет FK,
  не pin-ит raw/revision и не считается storage reference. Checker под track lock
  проверяет существование subject и его принадлежность track до записи issue. Пока
  track существует, после purge raw/revision historical row может сохранять UUID
  удалённого subject, не блокируя cleanup. Physical purge самого track удаляет все
  его active и resolved issue rows вместе с derived data.
  Отдельный DB CHECK требует
  `subject_type <> 'track' OR subject_id = track_id`. Новый subject type добавляется
  migration-ой CHECK и одновременно в закрытый application keyword set.
  `has_problems` не является отдельным mutable boolean: это derived flag наличия хотя
  бы одной active row. Из issue-related данных обычная owner projection содержит
  только этот flag и generic warning; administrator получает список всех active
  issue records. Для full track lock вся owner projection ограничена отдельным
  field allowlist ниже.
- Повторный detection той же active identity идемпотентно сохраняет первоначальный
  `detected_at`, обновляет `last_seen_at` и admin detail без новой row. После
  resolution повторное появление создаёт row с новым ID и timestamps как отдельный
  incident. Ни одно изменение issue state не создаёт user notification, web-inbox
  record или messenger delivery. Owner при чтении track видит только текущее
  `has_problems` и generic warning; resolved history остаётся admin-only.
- Каждая issue-state transaction сначала блокирует соответствующую `tracks` row
  через `SELECT ... FOR UPDATE`, затем читает и меняет `track_issues`. Partial unique
  index остаётся DB backstop от duplicate active identity. Отдельной issue operation
  owner lock не нужен; если более широкая operation уже должна блокировать owner,
  она соблюдает порядок `user -> track` и никогда не берёт `user` после `track`.
  PostgreSQL advisory locks не используются.
- Commit issue-state transaction является linearization point активации full track
  lock. Canonical single-track detail/download координируются с ней конфликтующим
  `SELECT ... FOR SHARE`, а content/lifecycle mutations — существующим
  `SELECT ... FOR UPDATE`; active issues повторно проверяются под track lock, который
  удерживается до формирования response decision, включая presigned URL, либо до
  commit mutation. Если content operation получила lock первой, detector transaction
  ждёт и operation завершается как pre-lock; если issue commit произошёл первым,
  следующая operation видит effective block. Search/map/catalog queries не берут
  locks на каждую track row и используют обычный PostgreSQL statement snapshot:
  collection query, начатый до issue commit, может завершиться со старым result.
  Уже начатые HTTP/S3 responses не отменяются.
- Scan вне issue-state transaction только находит candidate. Operation после захвата
  `tracks FOR UPDATE` выполняет финальный authoritative detector-specific check и
  только по его результату upsert-ит либо resolve-ит issue в той же transaction.
  Timeout/error откатывает transaction и оставляет issue state неизменным; результат
  предварительного scan никогда сам не меняет PostgreSQL. Под lock выполняется ровно
  одна check attempt с hard deadline 10 секунд, без retry, sleep или backoff;
  transient network/timeout/`5xx` откатывает всю operation, а повтор начинается
  позже с новой transaction и новым track lock. Отдельных retry rows, Valkey Stream
  или background scheduler для detector/recheck в MVP нет. Admin recheck показывает
  safe error; повтор запускает operator/administrator явно. Failed attempt фиксируется
  только в structured logs/metrics и не создаёт durable retry state.
- Authoritative detector-specific check возвращает ровно один из трёх результатов.
  `problem_present` является conclusive success: новая issue создаётся либо active
  row сохраняет `detected_at` и обновляет `last_seen_at`/safe admin detail.
  `healthy` ставит active row `resolved_at`. `inconclusive` означает, что invariant
  доказать нельзя; transaction откатывается без изменения issue. Ни один результат
  не создаёт user notification.
- Для S3 exact-key `404`/`NoSuchKey` и успешно завершённое чтение с SHA-256 либо size
  mismatch являются `problem_present`; совпавшие expected invariants — `healthy`.
  `401`/`403`/`429`, DNS/TLS/network/timeout, `5xx`, truncated body и неожиданный
  provider response являются `inconclusive`, вызывают operational alert и rollback
  без track issue mutation. Ошибка доступа к storage не маскируется как проблема
  конкретного track.
- Raw object получает `healthy` только после streaming GET полного body с
  одновременным пересчётом SHA-256 и byte size против `raw_objects`; весь object в
  memory не буферизуется. `HEAD` может выбрать missing candidate, но не resolve-ит
  integrity issue. S3 `ETag` и custom object metadata не являются доказательством
  exact original bytes и не заменяют private stored digest. Reader имеет hard cap
  stored `byte_size + 1`: первый лишний byte даёт conclusive
  `problem_present`/size mismatch, stream немедленно закрывается и остаток object не
  скачивается. Clean, корректно framed EOF до expected size также является
  conclusive `problem_present`/size mismatch. Connection abort, broken chunk
  framing, premature EOF относительно declared `Content-Length` и иная transport
  truncation являются `inconclusive`. Только clean EOF ровно на expected size
  допускает сравнение SHA-256.
- `resolved_at` меняет только detector-specific successful recheck. Administrator
  может инициировать recheck и добавить append-only audit note, но не force-clear
  issue. Storage checker проверяет фактическое появление и integrity/metadata object,
  другие codes — соответствующий validator. Failed check ничего не закрывает;
  resolved row сохраняется как history до physical purge owning track. Purge удаляет
  active и resolved issues, не устанавливая им `resolved_at`; issue rows не являются
  retention pin и не блокируют purge. Отсутствие user notifications не создаёт
  альтернативного manual resolution path.
- Successful recheck последней active full-lock issue автоматически пересчитывает
  effective capability block. Resolution меняет только `track_issues.resolved_at` и
  не меняет `tracks.status`, `current_revision_id` или moderation records. Если
  текущий lifecycle state разрешает доступ, прежний immutable snapshot снова
  доступен без новой moderation/publication operation. Archive или moderator
  removal, выполненные во время lock, сохраняются; другие active issues продолжают
  свои блокировки.
- Наличие корректно сохранённого track marker не блокирует global startup: сервис
  может работать в degraded mode, а затронутые операции этого track обязаны
  fail-safe. Incomplete integrity scan и нарушение без однозначного durable track
  target остаются global startup blockers.
- Marker не является blanket lock. Reason code детерминированно задаёт blocked
  capabilities, а effective block является union mappings всех active
  `track_issues`; отдельный mutable scope в row отсутствует.
  Unaffected reads и mutations остаются доступны. `raw_object_missing` и
  `raw_integrity_mismatch` блокируют re-parse, `sanitized_export_missing` —
  download/publication switch, а metadata view остаётся доступным.
  `snapshot_integrity_unknown` даёт full track lock: normal content serving,
  owner/content и publication mutations, restore, approve и другие
  content-dependent moderation decisions fail-closed блокируются. Разрешены только
  minimal status projection с generic warning, owner archive и moderator
  removal/hide как containment, scheduled purge, admin issue/history read,
  detector-specific recheck/resolution и append-only audit. Containment не читает и
  не публикует snapshot; owner warning не раскрывает codes или blocked capabilities.
  Owner-facing minimal projection содержит только stable `track_id`, текущий
  lifecycle `status`, derived `has_problems = true`, `purge_at` для уже archived
  track, локализованный generic warning и доступное owner containment-действие
  «Архивировать». UI использует generic title с коротким track ID и не читает
  revision. Name/Description, geometry, metrics, POI, classification, author,
  `current_revision_id`, export/raw IDs, issue code/detail/count/timestamps и список
  blocked capabilities отсутствуют. Для archived full-locked track restore скрыт;
  показывается только purge deadline. Administrator использует отдельный
  issue/history view.
  Owner mutation, blocked active full track lock, завершается terminal
  `409 Conflict` с `Cache-Control: no-store`. Stable JSON body равен ровно
  `{"error":"track_temporarily_unavailable"}`; htmx/HTML получает локализованный
  generic error partial. Private-chat bot не выполняет mutation, отвечает нейтральным
  callback acknowledgement и обновляет сообщение до minimal card только с доступным
  archive action. Domain rejection не retry-ится и не попадает в DLQ. `423 Locked`
  не используется; `503` остаётся public content/infrastructure response.
  `Retry-After`, issue code/detail/reason и blocked capability list отсутствуют.
  Principal с moderation permission, но без admin issue permissions, видит
  full-locked track в существующих moderation surfaces как snapshot-free placeholder;
  отдельная quarantine queue не создаётся. Placeholder содержит только `track_id`,
  lifecycle `status`, `has_problems = true` и generic warning. Единственное
  moderator action — существующее removal/hide с обязательной причиной. Preview,
  approve, `changes_requested`, export retry, download и все content/revision fields
  скрыты. Issue code/detail/history и detector recheck
  доступны только через отдельные admin permissions в issue/history view; наличие
  admin role/permissions не обходит full lock для content operations.
  Stale или напрямую отправленная moderator content mutation после обычных
  authentication, permission и resource-visibility checks проходит capability gate
  и получает тот же terminal `409 Conflict`, `Cache-Control: no-store` и JSON
  `{"error":"track_temporarily_unavailable"}` либо generic HTML. Domain/moderation
  mutation и audit decision не создаются; UI обновляется до placeholder. `403`
  используется только при отсутствующей permission, `404` — для невидимого или
  удалённого resource, `503` — для infrastructure failure. Capability rejection не
  retry-ится и не попадает в DLQ; issue details не раскрываются.
  Успешный moderator removal/hide как full-lock containment создаёт установленную
  locked-on owner notification в той же transaction, что lifecycle decision,
  moderation audit и outbox intent. Это notification удаления, а не issue-state
  notification; оно не раскрывает наличие или причину full lock.
  Search, map и catalog collection results исключают такой track. Прямые canonical
  detail и download URLs для otherwise published track отвечают generic
  `503 Service Unavailable` с `Cache-Control: no-store`, без `Retry-After`, track
  metadata, issue code/detail или причины. Private, archived и moderator-removed
  resources по-прежнему не различаются внешним `404`. После resolution следующий
  request заново проходит обычную publication/capability проверку; failure response
  не кэшируется.
  Full track lock является временным capability quarantine, а не переводом export из
  current/public в non-public: сам lock не удаляет и не ротирует sanitized object и
  не добавляет download proxy. Уже выданный presigned URL может работать до конца
  исходного five-minute TTL, а уже закэшированный GeoJSON — до 90 секунд
  (`max-age=30` плюс `stale-while-revalidate=60`). Уже доставленный client content
  не отзывается. После resolution тот же неизменённый snapshot и sanitized object
  снова обслуживаются, если lifecycle state по-прежнему разрешает публикацию.
- Chat-представление «Мои треки/черновики» объединяет принадлежащие пользователю
  upload/draft flows и tracks в один список с нормализованными статусами. Элементы,
  требующие действия owner, сортируются первыми; остальные — по последнему изменению.
- Full-locked entry в owner list заменяет обычную revision-backed карточку на
  установленный minimal status projection с generic title/short track ID; никакие
  revision fields для её построения не читаются.
- User-archived tracks не входят в default list и доступны через filter «Архив».
  До `purge_at` карточка показывает оставшийся срок и позволяет восстановление;
  отдельной операции досрочного permanent delete в MVP нет.
- Status filters кроме default «Все» и «Архив» отсутствуют в MVP.
- User archive сохраняет pre-archive status/current revision. Restore изменяет тот же
  `track_id` и до `purge_at` возвращает сохранённое состояние без создания revision:
  прежний approved snapshot снова публикуется, private track остаётся private.
  Moderator removal является отдельным lifecycle state и не eligible для user restore.
- Restore не создаёт отдельный confirmation state. Один valid owner callback
  атомарно проверяет `archived`/`purge_at` и меняет lifecycle state; повторная
  доставка той же операции идемпотентна.
- Chat actions проверяют текущий durable status перед mutation. `pending_review`
  доступен owner только для просмотра; исправление `changes_requested` создаёт новую
  immutable revision. Удаление draft и архивация track выполняются только после
  отдельного подтверждения.
- Chat list pagination возвращает по 10 entries и использует server-side keyset с
  deterministic tie-breaker; callback state не раскрывает entry IDs или cursor
  provider-у и привязан к requester и конкретному сообщению.
- Исходный `expires_at` chat list state равен 15 минутам от открытия и не меняется при
  переходах между страницами. Expired или потерянный ephemeral state не
  восстанавливается и не позволяет изменить старое сообщение.
- Geometry, S3 references, name/descriptions, activity, difficulty, season, duration
  и tags находятся в immutable full snapshot `track_revisions`.
- `track_revisions.base_revision_id UUID NULL` закрепляет moderation/diff baseline.
  Для первой публикации он равен `NULL`; revision изменения published track обязана
  ссылаться на `tracks.current_revision_id` в момент своего создания. Composite FK
  `(track_id, base_revision_id) -> track_revisions(track_id, id)` гарантирует ту же
  track lineage, а DB CHECK запрещает self-reference. Diff всегда строится по этой
  immutable baseline, а не по текущему pointer во время просмотра.
- `track_revisions.correction_of_revision_id UUID NULL` отличает авторский resubmit
  после `changes_requested` от обычной новой revision. Для resubmit поле обязательно
  и указывает на непосредственно возвращённую immutable revision того же track с
  outcome `changes_requested`; для остальных revisions оно равно `NULL`. Composite
  FK `(track_id, correction_of_revision_id) -> track_revisions(track_id, id)`
  гарантирует same-track lineage, self-reference запрещён. Resubmit сохраняет
  `base_revision_id` возвращённой revision.
- `track_revisions.submitted_for_review_at timestamptz NOT NULL` устанавливается
  атомарно при submit immutable revision в `pending_review` и после этого не
  изменяется. Это единственный age key обычной moderation queue.
- Private draft доступен только owner/moderators. Отправка создаёт `pending_review`.
- `pending_review` deactivated owner не меняет status, но не попадает в moderator
  queue. Approve/publish всегда повторно проверяет active owner; после реактивации item
  автоматически снова eligible для очереди без resubmit, когда его
  `export_state = ready`.
- Immutable pending revision до approval получает private sanitized export и durable
  `export_state` (`pending`, `ready`, `error`, `discarded`). Approve/publish разрешён
  только при `export_state = ready`; та же PostgreSQL transaction под обычным lock
  order переключает revision/track в `published` и обновляет `current_revision_id`.
  Generation failure оставляет revision в `pending_review`, старый current остаётся
  public, а export job проходит retry/DLQ. Отдельного публичного `publishing` status
  нет.
- До approval private sanitized object доступен только backend workers. Owner и
  moderator UI не получают presigned URL и не имеют отдельного authenticated
  download route. Download появляется после publication через canonical
  current-revision route.
- Обычная track moderation queue содержит только `pending_review` revisions active
  owners с `export_state = ready`. `pending|error` в неё не входят: `pending` остаётся
  техническим progress, а `error` после исчерпания automatic retries показывается в
  отдельном operator/admin operations view с idempotent retry и alert. Moderator не
  видит export-state infrastructure badges и не запускает export retry. Owner во всех
  состояниях до content decision видит только обычный `pending_review` без
  infrastructure details. Snapshot-free full-lock containment surface, описанный
  выше, не является обычной review queue и сохраняет только removal/hide независимо
  от export readiness.
- Обычная moderation queue общая и не назначает item конкретному moderator-у:
  `assigned_to`, claim table/lease, heartbeat, hand-off и отдельное состояние
  «в работе» отсутствуют. Любой moderator с нужной permission может открыть item.
  Decision transaction под установленными row locks повторно проверяет queue
  eligibility и все outcome-specific invariants. Первое успешно committed решение
  сохраняет actual moderator actor в audit; конкурентный submit после повторной
  проверки получает deterministic conflict с требованием обновить item и не создаёт
  второй decision, audit event или notification.
- Среди currently eligible items queue строго сортируется по
  `submitted_for_review_at ASC, track_revisions.id ASC` и использует эти же два поля
  для keyset pagination. Priority field, ручной bump/escalation, SLA buckets и особый
  приоритет resubmit отсутствуют. Full-lock containment и operator export-error view
  являются отдельными surfaces и в FIFO не входят.
- Все moderation decisions выполняются строго по одной revision. Queue/review UI не
  содержит checkboxes или «выбрать все»; bulk endpoints, массивы revision IDs и batch
  decision jobs отсутствуют для approve, `changes_requested` и moderator removal.
  Каждый request создаёт не более одной outcome transaction, одной audit chain и
  связанных с этой revision notifications. После успешного решения UI возвращается
  в обновлённую общую FIFO queue; конкурентный conflict использует тот же refresh.
- Approve и `changes_requested` не имеют дополнительного confirmation step. Approve
  выполняется явной primary submit-кнопкой без modal; отправка заполненной формы
  correction `reason_code`/`reason_note` сама подтверждает `changes_requested`.
  Submit control блокируется на время request, но correctness обеспечивается
  повторной backend-проверкой eligibility/invariants. Moderator removal сохраняет
  отдельный confirmation, поскольку немедленно скрывает публикацию и запускает
  90-дневный retention/appeal lifecycle.
- На обычном review-экране постоянно видимы только «Одобрить» как primary action и
  «Запросить изменения» как secondary action, раскрывающий inline correction form.
  Moderator removal вынесен из обычного action row в явно destructive danger-секцию
  и затем проходит отдельный confirmation. Snapshot-free full-lock containment
  surface является исключением: там removal — единственное доступное
  content-lifecycle действие.
- Для обычной track moderation нет durable состояний `deferred`, `snoozed` или
  `needs_second_review`, соответствующих columns/tables, filters, audit events и
  notifications. Закрытие review или возврат в queue не выполняет mutation: revision
  остаётся `pending_review` с прежним `submitted_for_review_at`/FIFO position и
  доступна любому moderator-у. Действие owner-а запрашивается через
  `changes_requested`, terminal case обрабатывается moderator removal; отсутствие
  решения не является domain event.
- Ordinary pending item не имеет внутренних moderator comments/thread до outcome:
  таблица `moderation_comments`, comment drafts/replies, mentions, attachments и
  unread state отсутствуют. До решения moderator не сохраняет mutable item data;
  moderation audit создаётся только вместе с outcome. Correction code/note является
  частью `changes_requested`, removal note — частью removal audit. Append-only admin
  notes для `track_issues` остаются отдельным operations-механизмом и не показываются
  в ordinary review.
- Ordinary moderation queue имеет одну FIFO list только из eligible
  `pending_review` items и keyset controls «Назад»/«Далее». Filters по
  first/update/resubmit, author/activity/reason, full-text search, saved views/queries
  и per-filter counts отсутствуют. Full-lock containment и operator export-error view
  остаются отдельными surfaces, а не filters общей queue.
- Queue row является только лёгкой navigation summary: track name, локализованный
  revision kind (`first`, `update`, `correction`) и submission timestamp/age.
  Единственное действие row — открыть review. Map thumbnail, geometry overlay, field
  diff, correction note, duration/duplicate hints и approve/changes/removal controls
  в list отсутствуют и загружаются только на review page. Queue query читает только
  нужные scalar columns и не загружает geometry или другие тяжёлые snapshot data.
- Moderation queue и уже открытый review не используют WebSocket, SSE, auto-polling
  timer или presence updates. UI обновляется только request-driven: при keyset
  navigation, manual reload и после decision/conflict. Открытый review может
  устареть; outcome transaction всё равно повторно проверяет state под locks и при
  проигранной race возвращает deterministic conflict со свежей queue. Фоновое
  удаление row или автоматическая перерисовка review отсутствуют.
- Проигравшая competing moderator mutation отвечает `409 Conflict`,
  `Cache-Control: no-store` и stable JSON
  `{"error":"moderation_item_already_decided"}`. Htmx/web показывает localized
  «Материал уже обработан другим модератором» и свежую FIFO queue. Winner actor,
  committed outcome/reason и retry action не раскрываются. Проигравший request не
  создаёт decision, audit event, notification или outbox intent.
- После auth/permission checks HTML GET уже обработанного ordinary item по stale
  direct link отвечает `303 See Other` на общую FIFO queue с одноразовым localized
  notice «Материал больше не доступен для проверки». Ordinary moderation не имеет
  historical read-only review mode и не раскрывает actor/outcome/reason. Полная
  moderation audit/history доступна только в отдельном admin view с собственной
  permission.
- Approve, `changes_requested` и moderator removal — append-only immutable outcomes.
  Ordinary moderation не имеет undo, reopen или edit decision; outcome и его
  reason/note не обновляются и не удаляются. Исправление после approve создаётся
  новой owner revision либо отдельным moderator removal, после `changes_requested`
  — новым author resubmit. Removal пересматривается только отдельным appeal/admin
  lifecycle. Любая корректировка добавляет новый audit event и не переписывает
  исходный decision.
- В MVP нет встроенной owner appeal form, `appeals` table, appeal status,
  attachments/chat thread или отдельной appeal queue. Owner обращается по
  `TRAILBASE_SUPPORT_URL` из removal notification, указывающей short track ID и
  90-дневный deadline. Support проверяет обращение out-of-band; результат оформляется
  отдельной permissioned admin operation с новым append-only audit event.
