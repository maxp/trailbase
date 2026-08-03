# Контракт: moderation и removal appeals

Часть [Implementation Contract](../IMPLEMENTATION-CONTRACT.md). При противоречии с другими документами действует этот контракт.

- Appeal management entrypoint содержит одно exact lookup field, принимающее полный
  track UUID либо short track ID из notification/support case. Списка/recent cases,
  queue, filters, fuzzy search по названию или owner-у и saved cases нет. Sensitive
  context загружается только для единственного точного совпадения и только после
  permission/fresh-auth checks. При collision short ID интерфейс не выбирает запись и
  требует полный UUID; unknown и ambiguous lookup дают одинаково нейтральный результат
  без sensitive context.
- После lookup case screen является одной read-only removal summary, а не второй
  moderation workspace. Он переиспользует historical snapshot renderer и показывает
  short/full track ID, current lifecycle state, original removal time, исходные
  reason code/note, purge deadline, immutable snapshot исходного decision и отдельный
  restore target: прежний approved snapshot либо явный private/editable branch без
  публикации. Computed restore eligibility показывается вместе со stable blocking
  class. Live owner/provider identity, editable track fields, support text/attachments
  и полный audit timeline не копируются; support verification остаётся out-of-band,
  новый appeal case record не создаётся.
- Case projection содержит один derived, не persisted `action_state` из закрытого
  каталога `decision_ready`, `restore_blocked_full_track_lock`,
  `restore_blocked_export_unavailable`, `window_closed`, `not_current`,
  `already_decided`. Precedence: `already_decided`, `not_current`, `window_closed`,
  full-track lock, retained export unavailable, `decision_ready`.
  `decision_ready` показывает обе outcome buttons; restore-blocked states оставляют
  uphold и показывают disabled restore с coarse localized reason; terminal states не
  показывают decision form, а `already_decided` показывает committed outcome. Payload
  не раскрывает issue codes/details. Backend recompute-ит state под track lock перед
  commit; submit conflict перезагружает summary и не доверяет прежнему UI state.
- Appeal management flow живёт только в HTML namespace `/admin/track-appeals`, отдельно
  от ordinary `/moderation/...`. `GET /admin/track-appeals` показывает exact lookup и
  optional query parameter `track_ref`, затем ту же read-only summary. Обе outcome
  buttons отправляют одну mutation route
  `POST /admin/track-appeals/:removal_decision_id/decision` с closed `outcome`,
  `reason_code`, optional `reason_note` и `idempotency_key`. POST требует active
  admin session, fresh auth и session-bound CSRF. UI htmx-enhanced с обычным full-page
  fallback; отдельных JSON/bot endpoints и `/moderation/...` aliases нет. Все
  responses используют `Cache-Control: no-store`.
- Если fresh auth истекла между render actionable form и Decision POST, ветка
  завершается до sensitive context reload и любой mutation отдельно от
  `422/409/503`: appeal outcome, audit, outbox и notification не создаются. Re-auth
  flow хранит server-side только allowlisted internal return target на canonical
  `GET /admin/track-appeals` с полным track UUID в query parameter `track_ref`;
  submitted `outcome`, `reason_code`, `reason_note` и прежний `idempotency_key` не
  переносятся в URL, auth-flow или session. После successful re-auth canonical GET
  заново вычисляет summary/`action_state`, создаёт новый `idempotency_key`, а admin
  повторно выбирает outcome и вводит reason. Open redirect и восстановление stale
  form отсутствуют.
- Successful re-auth записывает в active session только generic one-time flash kind
  `appeal_form_discarded`, без track/removal ID, outcome, reason или idempotency key.
  Canonical appeal GET атомарно consume-ит flash и показывает localized coarse notice
  «Предыдущая отправка не была сохранена. Проверьте актуальное состояние и выберите
  решение заново». Flash отсутствует в query, не расширяет error-code catalog
  `409/503` и исчезает после одного render. Authoritative summary имеет приоритет:
  terminal `action_state` не показывает form независимо от notice.
- Expired-fresh-auth navigation имеет один user flow с двумя HTTP representations.
  Обычный full-page Decision POST отвечает `303 See Other` на server-generated
  fresh-auth start URL. Htmx POST отвечает `200 OK`, `Cache-Control: no-store` и
  `HX-Redirect` на тот же URL с пустым body, поскольку htmx не обрабатывает response
  headers на `3xx`; оба варианта выполняют top-level full-page navigation. Они
  используют один server-side `return_to` на canonical appeal GET и не рендерят auth
  UI внутри appeal fragment. Submitted body уже отброшен. Invalid session и invalid
  CSRF остаются отдельными fail-closed ветками и не маскируются как expired fresh auth.
- Appeal browser re-auth переиспользует обычный bot-issued `web_session` token и
  существующий `/auth` GET/POST flow. Successful consume rotate-ит current browser
  session, переносит исходный `fresh_authenticated_at` из token record и возвращает по
  bound `return_to`; отдельного appeal/re-auth credential или consume route нет.
- Link для этой ветки выпускается непосредственно explicit private-chat action
  «Подтвердить вход». Только bound validated command/callback создаёт freshness;
  недавние сообщения, notifications, background events и browser activity её не
  создают, дополнительного PIN или второго confirmation нет.
- Admin может выполнить это действие через любую active linked Telegram/Max identity.
  Token bound к exact identity/provider/user/event, consume повторно проверяет active
  membership; unlinked/foreign token terminal invalid, а successful freshness
  становится account-level. Primary provider не получает отдельной auth роли.
- Same-browser consume при valid current session того же user является credential
  refresh и не создаёт `new_session` notification; новые token/CSRF/freshness заменяют
  current session, остальные sessions не затрагиваются. Без same-user current session
  auth commit создаёт provisional ordinary login, а locked-on security notification —
  только его claim.
- При таком refresh session сохраняет original `created_at`, получает new token/CSRF,
  bot-derived freshness и current `last_seen_at`/safe User-Agent summary. One-year
  sliding TTL начинается после provisional claim; отдельного `reauthenticated_at` нет,
  ordinary new login создаёт новый `created_at`.
- При render actionable form server генерирует 128-bit CSPRNG base64url
  `idempotency_key`; обе outcome buttons отправляют одно hidden field, отдельное от
  session-bound CSRF. Committed outcome хранит только
  `idempotency_key_hash bytea NOT NULL UNIQUE = SHA-256(raw key)`; raw key, отдельная
  idempotency table, TTL и cleanup job отсутствуют. Retry того же hash сравнивает exact
  normalized payload (`removal_decision_id`, `outcome`, `reason_code`, `reason_note`):
  полное совпадение возвращает committed result, любое отличие отвечает
  `409 Conflict`, `Cache-Control: no-store`, `idempotency_key_reused`. Новый key после
  уже committed outcome получает `appeal_already_decided`. Key не является auth secret,
  но ни raw value, ни hash не логируются.
- После successful first commit или exact idempotent retry full-page POST отвечает
  `303 See Other` на `GET /admin/track-appeals` с canonical full track UUID в query
  parameter `track_ref`; htmx выполняет client navigation к тому же GET, не получает
  отдельный success fragment. Final summary имеет `action_state = already_decided`,
  показывает committed outcome и одинаковый status «Решение сохранено» для commit и
  retry. Refresh не повторяет POST, canonical success URL не использует short ID.
  Validation error остаётся на form, domain/idempotency conflict загружает fresh
  authoritative summary.
- Decision POST использует одну HTML error matrix. `422 Unprocessable Content`
  rerender-ит form с field errors, введёнными values и тем же `idempotency_key`; outcome,
  audit/outbox не создаются. `409 Conflict` отбрасывает stale form/key и возвращает
  fresh summary/`action_state` с coarse localized notice и stable error code; branch не
  создаёт side effects. `503 Service Unavailable` сохраняет values и тот же key только
  для explicit manual retry без automatic retry/polling; response не утверждает, был ли
  outcome committed при uncertain COMMIT, и same-key retry это разрешает. Все ветви
  используют `Cache-Control: no-store`, не раскрывают stack trace, issue details или raw
  provider/storage errors; htmx и full-page имеют одинаковую semantics.
- Error-code catalog для этой matrix закрыт. `409` использует только
  `appeal_already_decided`, `appeal_window_closed`, `appeal_not_current`,
  `appeal_not_restorable` и `idempotency_key_reused`; `503` — только
  `appeal_temporarily_unavailable`. Первые три `409` выбирают summary соответственно с
  `already_decided`, `window_closed` и `not_current`; `appeal_not_restorable`
  возвращает заново вычисленный `restore_blocked_full_track_lock`,
  `restore_blocked_export_unavailable` либо `decision_ready`, а
  `idempotency_key_reused` — fresh authoritative summary того же case. `503` не
  утверждает authoritative `action_state` или результат commit. HTML template
  выбирается только по HTTP status и code и показывает локализованный coarse text;
  free-form backend message, exception/SQL/storage/provider names и issue codes/details
  в response не попадают.
- Appeal/admin operation имеет ровно два terminal outcome: `uphold_removal` и
  `restore_after_appeal`. Первый не меняет lifecycle state, второй является отдельной
  state-changing admin operation. Оба ссылаются на исходный removal decision, требуют
  обязательную admin reason, создают новый append-only audit event и не изменяют
  исходный removal. Generic `resolved`, partial outcome и custom/free-form outcome
  отсутствуют.
- Оба outcome требуют permission `track_appeal_decide`, mapped только к фиксированной
  роли `admin`, и existing fresh auth. Uphold и restore не разделяются на две
  permissions. Operation доступна только в management UI; ordinary moderator role,
  moderation endpoints и bot action её не получают. Permission/fresh-auth checks
  выполняются до загрузки sensitive audit context, committed audit сохраняет actual
  admin actor.
- Final submit заполненной outcome+reason form является единственным confirmation для
  appeal decision; второго modal или отдельного confirm screen нет. Fresh-authenticated
  management form показывает short track ID, current state, выбранный outcome, его
  последствия и обязательную reason. Две визуально разделённые outcome-specific кнопки
  называются «Подтвердить удаление» и «Восстановить трек»; generic «Сохранить»
  отсутствует. Controls disabled in-flight, а backend перед commit повторно проверяет
  permission, fresh auth, single-shot и lifecycle invariants. Retry следует общей
  idempotency semantics appeal outcome.
- Appeal form не показывает live fresh-auth countdown или exact expiry timestamp.
  Рядом с decision controls есть только localized static hint «Подтверждение входа
  действует 10 минут; если оно истечёт, потребуется подтвердить вход снова».
  JavaScript timer, polling, auto-refresh и auto-submit отсутствуют; client clock не
  участвует в authorization. Authoritative freshness проверяется backend-ом при POST,
  а expiry использует принятые discard/re-auth/fresh-summary/one-time-notice ветки.
- Actionable render и Decision POST используют один server-side predicate
  `now < fresh_authenticated_at + 10 minutes` без UX safety buffer или второго
  threshold. При равенстве deadline и после него fresh auth считается expired. GET не
  показывает form, который POST уже заведомо отклонит по иному freshness margin; если
  deadline проходит во время заполнения, POST использует обычный discard/re-auth flow.
- Decision POST фиксирует один server `authorization_now` при authoritative mutation
  guard внутри transaction, непосредственно перед authoritative preconditions и
  первой mutation. Если в этой точке `authorization_now < fresh_authenticated_at + 10
  minutes`, operation атомарно завершается даже при physical commit после deadline;
  повторной wall-clock проверки на commit boundary нет. Queue/lock latency после
  успешного guard не превращает уже авторизованную operation в fresh-auth expiry.
- Appeal GET и POST получают `now` из одного injected application-level UTC clock с
  `java.time.Instant` semantics. Client clock и PostgreSQL `now()` в freshness
  authorization не участвуют; отдельного DB round trip ради времени нет. Все app
  instances обязаны иметь синхронизированные UTC system clocks, а DB-managed
  timestamps могут использовать PostgreSQL clock, не становясь вторым freshness
  authority.
- Исходный `fresh_authenticated_at` для appeal re-auth фиксируется тем же application
  clock при server-side validation/acceptance explicit bot webhook/callback. Timestamp
  provider-а не влияет на начало или длительность freshness и отбрасывается после
  validation входного payload; dedupe и exact event/identity bindings проверяются
  независимо от времени.
- `web_session` token/session auth records для appeal не содержат provider timestamp:
  только server `fresh_authenticated_at` и opaque event/identity bindings для уже
  принятой dedupe/consume validation. В MVP timestamp не создаёт также redacted
  operational event и не попадает в logs/metrics; остаются только server-time counters
  по provider/validation result. Измерение delivery delay можно добавить отдельным
  будущим контрактом, не меняя auth state.
- Appeal outcome переиспользует internal reason contract account reactivation:
  `reason_code text NOT NULL` с DB CHECK на `support_request_verified`,
  `administrative_correction`, `other`, без PostgreSQL enum. Optional
  `reason_note text` обязательна для `other`; если задана, после trim содержит 1–1000
  Unicode code points без control/newline characters. Code/note хранятся только в
  append-only admin audit, не попадают в owner notification или application logs.
  Validation и management UI общие с account reactivation; отдельного
  appeal-specific catalog нет.
- Appeal outcome single-shot для каждого исходного removal decision: persisted outcome
  хранит `removal_decision_id` под UNIQUE constraint, первый committed outcome
  побеждает. Retry с тем же idempotency key и тем же outcome возвращает сохранённый
  результат без второго audit/outbox. Конкурирующий либо последующий другой outcome
  получает `409 Conflict`, `Cache-Control: no-store` и stable
  `appeal_already_decided` без mutation или side effects. Reopen/second appeal для
  того же removal отсутствует; ошибочная admin decision исправляется только новой
  отдельной permissioned lifecycle operation, не переписывающей appeal outcome.
- `restore_after_appeal` не возвращает decided revision в ordinary moderation queue.
  Если до removal существовал approved `current_revision_id`, операция возвращает тот
  же immutable snapshot в published без новой ordinary moderation. Если approved
  revision не было, track становится private/editable; только явный owner resubmit
  создаёт новую immutable revision. Удалённая pending revision остаётся decided, её
  export — `discarded`, и она никогда повторно не становится queue-eligible.
- При moderator removal private sanitized export последнего approved
  `current_revision_id` сохраняется до `uphold_removal` либо 90-дневного purge как
  явное исключение из немедленного delete для non-public transition. Canonical
  detail/download всё время отвечают `404`, новые presigned URL не выдаются, object
  доступен только backend. `uphold_removal` и purge ставят idempotent delete;
  `restore_after_appeal` одной transaction возвращает тот же snapshot в published и
  записывает audit/outbox без regeneration job или промежуточного `restoring` state.
  Private export удалённой unapproved pending revision по-прежнему немедленно получает
  `discarded` и delete.
- `uphold_removal` сохраняет исходный `purge_at`: retained sanitized export удаляется
  сразу, но raw, revisions и остальные retained data очищаются только в первоначальный
  90-дневный deadline. Immediate purge, продление и новый retention deadline
  отсутствуют. `restore_after_appeal` в outcome transaction устанавливает
  `purge_at = NULL`, отменяя scheduled purge eligibility восстановленного track.
  Error/conflict/idempotent retry deadline не меняют. Appeal outcome и purge worker
  сериализуются track lock: purge-first делает restore non-restorable, restore-first
  заставляет worker повторно увидеть `purge_at = NULL` и завершиться без purge.
- Первый новый appeal outcome любого типа имеет общую base eligibility под track lock:
  исходный removal остаётся current/active, outcome ещё не сохранён и
  `now() < purge_at`. Expiration закрывает и `uphold_removal`, и
  `restore_after_appeal` даже при lag purge worker-а; out-of-band support request не
  продлевает окно. Newer lifecycle state также закрывает обе операции. До deadline
  active full-track lock и unavailable retained export являются restore-only blockers:
  они не запрещают audit-only uphold. Уже committed outcome остаётся доступен через
  прежнюю idempotent retry semantics, но нового решения после deadline нет.
- `restore_after_appeal` под track lock повторно требует
  `tracks.status = moderator_removed`, совпадение active removal с
  `removal_decision_id`, `now() < purge_at`, отсутствие более нового lifecycle event и
  active full-track lock, а для published branch — готовый retained export. Full lock
  блокирует restore, но не audit-only `uphold_removal`. Любой stale/blocked case
  отвечает `409 Conflict`, `Cache-Control: no-store` и stable
  `appeal_not_restorable` без appeal outcome, audit/outbox или object delete. После
  устранения временного block admin может повторить restore до deadline.
- Первый committed appeal outcome в той же transaction создаёт locked-on owner
  web-inbox record и primary-bot outbox intent. `uphold_removal` сообщает факт, short
  track ID и purge deadline без second-appeal action; `restore_after_appeal` даёт
  canonical link для published branch либо edit/resubmit action для private branch.
  Internal reason/note, admin actor, audit ID и context не раскрываются. Error/conflict
  и idempotent retry не создают новую notification; deactivated-account suppression
  применяется без backlog/replay.
- `changes_requested` или moderator removal pending revision одновременно ставит
  export lifecycle в `discarded` и пишет high-priority idempotent delete command
  private sanitized object. Revision и moderation audit сохраняются, object не
  переиспользуется: исправленная immutable revision после `changes_requested`
  получает новый export. Worker меняет только `pending -> ready`; late completion
  после discard не оживляет state и ставит созданный object на delete. Cleanup
  использует retry/DLQ и operator alert.
- Approve/publish и account deactivation берут owner row lock до track/revision locks
  (`user -> track -> revision`). Если approval commit-нулся первым, опубликованный
  snapshot остаётся public после деактивации; если первой commit-нулась deactivation,
  approval fail-closed не меняет revision/publication state.
- Первая публикация требует moderation. Обычная human moderation имеет только два
  исхода: approve/publish либо `changes_requested`; отдельного track outcome/state
  `rejected` нет. `changes_requested` decision и moderation audit требуют один
  `reason_code text NOT NULL` с DB CHECK на `metadata_correction`,
  `geometry_correction`, `privacy_correction`, `classification_correction`,
  `license_or_attribution`, `other`; PostgreSQL enum, array и checklist relation не
  используются. Optional `reason_note text` обязателен для `other`; если задан, после
  trim содержит 1–1000 Unicode code points без control/newline characters. Owner
  видит локализованный actionable label и note в locked-on web-inbox/primary-bot
  notification; note намеренно owner-visible и не попадает в logs. Несколько проблем
  moderator перечисляет в одном note. Correction catalog отделён от removal reasons
  и `track_issues.code`; значения между ними не копируются.
  Авторская правка создаёт новый immutable full revision и повторно отправляет его
  без обязательной загрузки GPX; changes-requested revision остаётся неизменным audit
  snapshot. По умолчанию moderator видит correction diff resubmit-а относительно
  его `correction_of_revision_id`, рядом — сохранённые correction code и note.
  Вторичными views остаются total diff относительно `base_revision_id` и полный
  pending snapshot.
  Terminal policy, privacy/safety, legal, spam и duplicate cases используют
  moderator removal с его confirmation, reason catalog и 90-дневным retention.
- Изменение опубликованного трека создаёт pending full revision. Публичная revision
  остаётся прежней до атомарного approval. Для первой публикации moderator review
  показывает полный snapshot. Для изменения опубликованного трека review по
  умолчанию показывает field diff и overlay geometry новой revision с её
  `base_revision_id`; отдельное действие открывает полный pending snapshot.
  Approve, `changes_requested` или moderator removal всегда применяются ко всей
  immutable pending revision. Field-level/partial approval отсутствует.
- Approval update revision под существующим track lock требует
  `tracks.current_revision_id = track_revisions.base_revision_id`. При mismatch
  mutation и moderation decision/audit не создаются, revision считается stale и не
  rebase-ится автоматически.
- Ownership transfer не входит в MVP. Будущий transfer требует audited operation и
  подтверждения обоих пользователей.
- Авторское «удаление» переводит track в `archived`; восстановление доступно 30 дней
  через filter «Архив». Досрочная физическая очистка пользователю не предлагается.
  Restore возвращает durable pre-archive status и current revision; повторная
  moderation неизменённого approved snapshot не требуется.
  После `purge_at` worker удаляет raw/export/photos, revisions, все active/resolved
  `track_issues` и остальные derived data, оставляя tombstone track UUID и audit без
  пользовательского содержимого. Codes/details/subject UUID удаляемых issues не
  копируются в новый audit event.
- Moderator removal скрывает публикацию сразу, не восстанавливается автором и хранит
  данные 90 дней для апелляции до такой же очистки. Решение требует
  `reason_code text NOT NULL` с DB CHECK на закрытый начальный каталог
  `policy_violation`, `privacy_or_safety`, `legal_request`, `spam`, `duplicate`,
  `technical_containment`, `other`; PostgreSQL enum не используется. Optional
  `reason_note text` хранится только в moderation audit и обязателен для `other`.
  Если note задан, после trim он содержит 1–1000 Unicode code points без control или
  newline characters. Note никогда не показывается owner-у и не пишется в logs.
  Owner-visible notification получает только локализованный public-safe label кода;
  `technical_containment` показывается как generic «Техническая недоступность».
  `track_issues.code` и `track_issues.detail` не копируются в removal record,
  notification или note.
- Автор может явно обрезать начало/конец маршрута с preview. Автоматической privacy
  обрезки нет. Опубликованная geometry соответствует одобренной trimmed revision.
- Повторный exact GPX того же пользователя вызывает warning, но может быть сохранён
  явно. Cross-user duplicate hints видит только moderator.
- Duplicate detection: hash canonicalized 2D geometry на grid `1e-6` градуса,
  direction-insensitive. Неидентичные кандидаты ищутся по bbox/length и
  `ST_HausdorffDistance`. Совпадение — только moderation hint.
