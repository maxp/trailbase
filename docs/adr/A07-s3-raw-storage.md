# ADR A07 — Хранение raw GPX и фото в S3-совместимом хранилище

**Status**: Accepted
**Date**: 2026-07-25

**Уточнение 2026-07-31:** transparent provider-side SSE или disk encryption raw
storage допустимы как deployment control. TrailBase не управляет provider keys или
key IDs и получает после GET те же exact original bytes без application decrypt
path. Для integrity хранится private SHA-256, вычисленный при streaming upload;
keyed HMAC и отдельный application secret не используются.

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
     opaque UUID key, затем async job создаёт private revision в PostGIS. Transparent
     provider-side SSE/disk encryption может быть включено на deployment: приложение
     не управляет его keys/key IDs и после GET получает исходные bytes. Raw доступен
     только backend workers для parse/re-parse; public/owner download route,
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
     exact `byte_size`, private SHA-256, lifecycle state и timestamps. Digest
     вычисляется при streaming upload, проверяется при последующих чтениях/re-parse,
     не публикуется и не используется для dedup или различимого поведения API.
     Mismatch уже `ready` raw обрабатывается fail-closed без нового lifecycle state:
     текущий parse/re-parse получает permanent `raw_integrity_mismatch` без automatic
     retry и без revision/publication mutation, operator alert содержит только
     `raw_object_id`, а row/object сохраняется по обычному retention для диагностики
     или восстановления. Следующая попытка снова проверяет digest; если
     восстановление невозможно, пользователь загружает source заново.
     Application/admin repair flow отсутствует. Infrastructure operator может
     восстановить под тем же opaque key только exact bytes с тем же stored SHA-256
     из MinIO versioning/off-host backup; digest не меняется. Без совпадающей копии
     пользователь делает новый upload/new `raw_object_id`, а другой source нельзя
     подставить в прежнюю raw row.
     После primary purge raw может оставаться только в encrypted operator-only
     off-host backup максимум 30 дней, недоступен приложению и не восстанавливается
     в active service. После PostgreSQL/WAL restore и migrations при остановленных
     web/workers operator запускает CLI reconciliation до открытия traffic.
     PostgreSQL является единственным источником references: любой
     TrailBase-managed S3 key без единой durable DB reference является orphan и
     удаляется со всеми versions/delete markers; referenced key не удаляется.
     Manifest/pending approval/reconciliation rows не хранятся.
     CLI default выполняет dry-run с counts/total bytes по storage class и nonzero
     exit при DB/S3 scan error. Удаление требует explicit destructive flag и нового
     полного scan при остановленных web/workers; dry-run result не сохраняется.
     Dry-run не меняет DB. Destructive run удаляет orphans и durable помечает все
     referenced-but-missing objects. Полный успешный scan/cleanup с однозначными
     track markers разрешает degraded startup; incomplete scan или unmappable
     violation оставляет traffic закрытым.
     Referenced-but-missing S3 object не удаляет DB reference: CLI ставит связанному
     track durable problem marker с admin-only reason code и безопасным пояснением.
     Administrator видит точную причину, owner — только нейтральный факт проблемы с
     записью. Тот же marker допускает другие track data-integrity problems.
     Reason блокирует только dependent capabilities: missing raw — re-parse, missing
     sanitized export — download/publication switch; исправные geometry/metadata
     остаются доступны. Full track lock используется только для snapshot-wide
     integrity reason. Он блокирует normal content serving, owner/content и
     publication mutations, restore, approve и content-dependent moderation.
     Разрешены только minimal status с generic warning, owner archive и moderator
     removal/hide как containment, scheduled purge, admin issue/history read,
     detector-specific recheck/resolution и append-only audit. Containment не читает
     и не публикует snapshot.
     Owner projection содержит только `track_id`, lifecycle `status`,
     `has_problems = true`, `purge_at` для archived track, localized generic warning
     и доступное действие «Архивировать». Generic title/short track ID строятся без
     revision read. Content/revision/export/raw fields, issue semantics и blocked
     capabilities отсутствуют; archived track показывает purge deadline без restore.
     Blocked owner mutation возвращает terminal `409 Conflict` с
     `Cache-Control: no-store` и JSON
     `{"error":"track_temporarily_unavailable"}` либо localized generic HTML. Bot
     отвечает neutral acknowledgement, не меняет state и обновляет minimal card;
     domain rejection не retry-ится/DLQ-ится. `423`, `Retry-After` и internal issue
     data отсутствуют; `503` остаётся public/infrastructure response.
     Moderator без admin issue permissions видит full-locked track в existing
     moderation surfaces как snapshot-free placeholder без новой queue: только
     `track_id`, lifecycle `status`, `has_problems = true`, generic warning и
     removal/hide с обязательной причиной. Preview/approve/changes-requested,
     export retry/download и content/revision fields скрыты. Admin issue permissions
     открывают отдельный history/recheck view, но не обходят content lock.
     Stale/direct moderator content mutation после auth/permission/visibility checks
     получает тот же terminal `409`/`no-store`/safe JSON error либо generic HTML без
     domain mutation или audit decision; UI возвращается к placeholder. `403`
     означает missing permission, `404` — invisible/removed resource, `503` — infra
     failure. Capability rejection не retry-ится/DLQ-ится и не раскрывает issue.
     Успешный removal/hide как containment атомарно создаёт обычный moderation audit,
     locked-on web-inbox record и primary-bot outbox для active owner. Notification
     содержит только removal fact, локализованный public-safe label закрытого
     `reason_code`, short track ID, `TRAILBASE_SUPPORT_URL` и 90-day purge/appeal
     deadline. Code хранится как `text` с DB CHECK
     на `policy_violation`, `privacy_or_safety`, `legal_request`, `spam`, `duplicate`,
     `technical_containment`, `other`, без PostgreSQL enum; containment показывается
     generic label «Техническая недоступность». Optional `reason_note` обязателен для
     `other`, хранится только в audit, после trim содержит 1–1000 Unicode code points
     без control/newline и не показывается owner-у или в logs. Full lock,
     `track_issues.code/detail`, snapshot и internal data не копируются.
     Detector/recheck notification не создают; deactivated-account suppression
     действует без backlog/replay.
     Search/map/catalog collection results исключают full-locked track. Прямые
     canonical detail/download URLs для otherwise published track отвечают generic
     `503 Service Unavailable` с `Cache-Control: no-store`, без `Retry-After`, track
     metadata или integrity details. Private/archived/moderator-removed resources
     сохраняют одинаковый `404`; после resolution следующий request заново проходит
     publication/capability check без cached failure.
     Full lock — capability quarantine, не non-public lifecycle transition: он сам
     не удаляет и не ротирует sanitized object. Уже выданный presigned URL может
     работать до исходных пяти минут, cached GeoJSON — до 90 секунд, а уже
     доставленный client content не отзывается. После resolution тот же immutable
     snapshot/export снова доступен, если lifecycle это разрешает; download proxy не
     добавляется.
     Несколько проблем хранятся отдельными `track_issues` rows с
     `code text NOT NULL`, admin-only `detail text NOT NULL` и detected/resolved
     timestamps. DB CHECK требует `char_length(detail) BETWEEN 1 AND 1000`;
     application формирует detail только из code-specific безопасных шаблонов без
     raw exceptions, HTTP headers/provider body, GPX content или credentials.
     Detail не использует `jsonb`, машинная семантика остаётся в `code`.
     Шаблоны образуют закрытый application-каталог с явным whitelist подстановок. В
     MVP разрешены только тип mismatch и точно известные expected/observed byte
     counts. Oversized read, остановленный на `expected + 1`, сообщает lower bound,
     а не точный observed size. `subject_id`, object key/bucket, filename, digest,
     provider response/status, exception и operator free text не подставляются;
     ручной текст администратора остаётся append-only audit note.
     Rendered detail хранится только на canonical English, независимо от `ru`/`en`
     UI locale, и не переписывается при её смене. Admin UI локализует labels/actions,
     но показывает stored detail как canonical diagnostic text.
     `code` имеет DB CHECK только на `raw_object_missing`, `raw_integrity_mismatch`,
     `sanitized_export_missing` и `snapshot_integrity_unknown`, без PostgreSQL enum;
     новый code требует migration существующего CHECK, checker, subject constraint
     и mapping. Отдельной scope column нет: blocked capabilities выводятся из
     закрытого code mapping. Derived
     `has_problems` означает наличие active rows; effective capability block является
     union mappings active codes.
     Initial mapping:
     `raw_object_missing|raw_integrity_mismatch -> reparse`,
     `sanitized_export_missing -> download+publication_switch`,
     `snapshot_integrity_unknown -> full_track_lock`. Unknown code отклоняется; новый
     требует migration, checker, subject constraint и mapping.
     `subject_id` является UUID NOT NULL: track-wide issue использует
     `track`/`track_id`, raw — `raw_object`/`raw_objects.id`, revision и sanitized
     export — `revision`/`track_revisions.id`. `subject_type` — text с DB CHECK
     `track|raw_object|revision`, без PostgreSQL enum; отдельный CHECK требует для
     track subject равенство `subject_id = track_id`. Compound CHECK допускает
     только `raw_object_missing|raw_integrity_mismatch` с `raw_object` и
     `sanitized_export_missing|snapshot_integrity_unknown` с `revision`; начальный
     code set не допускает `track`, а первый track-wide code миграцией расширяет code
     CHECK и pair CHECK. Stable active identity
     `(track_id, code, subject_type, subject_id)` защищена partial unique constraint
     без NULL/sentinel semantics. Повторный detection сохраняет `detected_at`,
     обновляет `last_seen_at`/detail; recurrence после resolution создаёт новый
     incident.
     `track_id` имеет FK, но polymorphic `subject_id` является historical UUID без
     FK, retention pin или storage-reference semantics. Checker под track lock
     проверяет subject membership; CHECK для `track` требует
     `subject_id = track_id`. Purge raw/revision не блокируется historical issue;
     пока track существует, row может хранить UUID удалённого subject. Physical
     purge track удаляет все active/resolved issue rows вместе с derived data: они
     не являются retention pin, не получают fake `resolved_at` и не копируются как
     codes/details/subject UUID в новый audit event. Tombstone track UUID и уже
     существующий append-only audit остаются.
     Storage issue получает `resolved_at` только после successful detector-specific
     recheck object/integrity metadata. Admin может запустить recheck и добавить
     append-only audit note, но не force-clear issue; row остаётся в history до
     physical purge owning track.
     Resolution последней active full-lock issue автоматически пересчитывает
     effective capability block и меняет только `track_issues.resolved_at`, не
     `tracks.status`, `current_revision_id` или moderation records. Допустимый
     текущим lifecycle прежний immutable snapshot снова доступен без новой
     moderation/publication operation. Archive/moderator removal и блокировки других
     active issues сохраняются.
     Отсутствие user notifications не создаёт другого resolution path.
     Изменения issue state не создают user notification, web-inbox record или
     messenger delivery. Из issue-related данных owner видит только текущий derived
     `has_problems` и generic warning в track views; codes/details/history остаются
     admin-only, а full-lock projection использует заданный выше field allowlist.
     Issue-state transactions сериализуются `SELECT ... FOR UPDATE` на `tracks` row.
     Если operation также нужен owner lock, применяется порядок `user -> track`;
     standalone issue operation блокирует только track. Advisory locks не
     используются, partial unique index остаётся DB backstop.
     Commit issue transaction linearize-ит activation full lock. Canonical
     single-track detail/download используют конфликтующий `FOR SHARE` до response
     decision/signing, mutations — `FOR UPDATE` до commit; active issues повторно
     проверяются под lock. Operation, получившая lock до detector transaction,
     завершается как pre-lock; после issue commit новые operations видят block.
     Search/map/catalog используют statement snapshot без per-track locks, поэтому
     начатый до commit collection query может вернуть прежний result. In-flight
     HTTP/S3 responses не отменяются.
     Preliminary scan только находит candidate; authoritative detector-specific
     check повторяется под `tracks FOR UPDATE` в изменяющей issue transaction.
     Timeout/error откатывает её без изменения issue state. Под lock выполняется
     одна attempt с hard deadline 10 секунд, без retry/backoff; следующая attempt
     всегда является новой operation с новым lock. Отдельных retry rows/stream/job
     нет: CLI/admin возвращают safe failure, explicit rerun выполняет
     operator/administrator, logs/metrics фиксируют attempt.
     Checker возвращает `problem_present`, `healthy` или `inconclusive`.
     Conclusive problem создаёт/обновляет active issue и `last_seen_at`; healthy
     ставит `resolved_at`; inconclusive откатывает transaction без issue mutation.
     S3 `404`/`NoSuchKey` и completed-read SHA-256/size mismatch являются
     `problem_present`; exact match — `healthy`; access/permission/`429`,
     DNS/TLS/network/timeout, `5xx`, truncated/unexpected response —
     `inconclusive` с operational alert, без track issue mutation.
     Raw `healthy` подтверждает только full streaming GET с пересчётом SHA-256/size
     по body против `raw_objects`, без full-body memory buffer. HEAD/ETag/custom
     metadata не resolve-ят integrity issue и не заменяют private digest. Read cap
     равен stored `byte_size + 1`: extra byte conclusive size mismatch и закрывает
     stream без скачивания остатка. Clean short EOF — conclusive size mismatch,
     transport abort/broken framing/premature EOF — `inconclusive`; SHA сравнивается
     только после exact-size EOF.
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
     `delete_pending` row, пагинированно перечисляет и удаляет по version ID все
     versions/delete markers exact key. Prefix matches других keys не удаляются;
     только empty exact-key listing разрешает physical row delete. Missing
     version/object/row и crash между S3/DB delete обрабатываются идемпотентным
     replay. Wrong state или retained FK fail-closed проходят retry/DLQ+alert;
     storage metadata в outbox не копируется.
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
     `publishing` status; job проходит retry/DLQ. Обычная moderator queue содержит
     только active-owner revisions с `export_state = ready`; pending/error badge и
     retry action там отсутствуют. Exhausted error попадает в отдельный operator/admin
     operations view с idempotent retry и alert, owner infra details не видит.
     Snapshot-free full-lock containment остаётся отдельным removal-only surface.
     Обычная queue общая для всех moderators с permission и не хранит assignment,
     claim lease, heartbeat, hand-off или состояние «в работе». Decision transaction
     повторно проверяет eligibility/invariants под row locks; первый commit сохраняет
     actual actor в audit, конкурентный submit получает deterministic conflict без
     второго decision, audit или notification.
     Submit immutable revision атомарно устанавливает неизменяемый
     `submitted_for_review_at timestamptz NOT NULL`. Eligible queue строго сортируется
     по `(submitted_for_review_at ASC, track_revisions.id ASC)` с keyset pagination;
     priority/escalation, manual bump, SLA buckets и resubmit priority отсутствуют.
     Containment и export-error surfaces в FIFO не входят.
     Approve, `changes_requested` и moderator removal обрабатывают по одной revision:
     checkbox/select-all UI, bulk ID endpoints и batch decision jobs отсутствуют.
     Каждый успешный request создаёт только per-revision audit/notifications и
     возвращает moderator-а в обновлённую общую FIFO queue; conflict делает тот же
     refresh.
     Approve выполняется primary submit-кнопкой без confirmation modal; submit
     заполненной correction code/note form сам подтверждает `changes_requested`.
     Controls disabled in-flight, backend re-check обязателен. Moderator removal
     сохраняет отдельный confirmation из-за немедленного hide и retention/appeal
     lifecycle.
     Ordinary review постоянно показывает approve primary action и
     `changes_requested` secondary action с inline correction form; removal вынесен
     в destructive danger-секцию. Full-lock containment остаётся removal-only
     исключением.
     Durable `deferred`, `snoozed` и `needs_second_review` states отсутствуют. Выход
     из review не создаёт mutation/audit/notification и оставляет revision в
     `pending_review` на прежней FIFO position, доступной любому moderator-у.
     Internal `moderation_comments`, drafts/replies, mentions, attachments и unread
     state отсутствуют. Moderator-authored data появляются только в outcome audit;
     admin `track_issues` notes остаются отдельным operations flow.
     Ordinary queue — одна FIFO list eligible `pending_review` items с keyset
     «Назад»/«Далее» без filters, search, saved views/queries или per-filter counts;
     containment и export-error operations остаются отдельными surfaces.
     Queue row содержит только track name, localized `first|update|correction` kind
     и submission time/age; единственное действие — открыть review. Geometry/diff/
     correction/hint previews и outcome controls загружаются только на review page,
     а list query не читает geometry или тяжёлые snapshot data.
     Queue/review не имеют WebSocket, SSE, auto-polling timer или presence updates.
     Refresh выполняется только navigation/manual reload/decision/conflict; stale
     review безопасно отклоняется authoritative transaction re-check и заменяется
     свежей queue.
     Race loser получает `409 Conflict`, `Cache-Control: no-store` и stable
     `moderation_item_already_decided`; UI показывает neutral notice и свежую queue
     без winner/outcome/reason, retry или второго audit/notification/outbox.
     Stale direct HTML GET обработанного ordinary item после permission checks делает
     `303` в FIFO queue с neutral one-time notice. Historical ordinary review
     отсутствует; audit/history доступна только в отдельном permissioned admin view.
     Approve, `changes_requested` и moderator removal — append-only immutable
     outcomes без undo, reopen, edit или delete decision в ordinary moderation.
     Исправление после approve создаёт новую owner revision либо отдельный removal,
     после `changes_requested` — новый author resubmit; removal пересматривается
     отдельным appeal/admin lifecycle. Исходный outcome и reason/note неизменны, а
     корректировка создаёт новый audit event.
     Встроенных owner appeal form, `appeals` table, appeal status,
     attachments/chat thread и отдельной appeal queue нет. Removal notification
     направляет owner-а по `TRAILBASE_SUPPORT_URL` с short track ID и 90-дневным
     deadline. Support verification выполняется out-of-band, а результат фиксируется
     отдельной permissioned admin operation с новым append-only audit event.
     Appeal management entrypoint содержит одно exact lookup field по полному track
     UUID либо short track ID из notification/support case. Recent cases, queue,
     filters, fuzzy owner/name search и saved cases отсутствуют. Sensitive context
     загружается только после permission/fresh-auth checks для unique exact match.
     Collision short ID требует полный UUID без automatic selection; unknown/ambiguous
     lookup возвращает одинаково нейтральный результат без context.
     Short track ID вычисляется из последних 12 lowercase hex UUIDv7 `tracks.id` и
     отображается как `trk-xxxx-xxxx-xxxx`. Отдельных column/sequence/generator и
     UNIQUE assumption нет. Lookup нормализует ASCII case, optional exact `trk-` и
     group hyphens, принимая ровно 12 hex либо canonical full UUID; non-unique
     expression index обслуживает short path, collision требует full UUID. Первые 8
     hex GPX filename остаются display-only suffix и lookup ID не являются.
     После lookup management UI показывает одну read-only removal summary через
     historical snapshot renderer, а не вторую moderation workspace. Summary содержит
     short/full ID, lifecycle state, original removal time, исходные reason code/note,
     purge deadline, immutable decision snapshot, точный restore target и computed
     eligibility со stable blocking class. Live owner/provider identity, editable
     fields, support text/attachments и full audit timeline не копируются; persisted
     appeal case не создаётся, support verification остаётся out-of-band.
     Case projection содержит один derived/non-persisted `action_state`:
     `decision_ready`, `restore_blocked_full_track_lock`,
     `restore_blocked_export_unavailable`, `window_closed`, `not_current`,
     `already_decided`. Precedence идёт от decided/not-current/window-closed к full
     lock, export unavailable и ready. Ready показывает обе кнопки, restore-blocked
     state — uphold плюс disabled restore с coarse localized reason, terminal states
     скрывают form; decided показывает committed outcome. Issue details отсутствуют.
     Backend recompute-ит state под lock, UI после conflict reload-ит summary и сам
     eligibility не вычисляет.
     Appeal management использует отдельный HTML namespace `/admin/track-appeals`.
     `GET /admin/track-appeals` обслуживает exact `track_ref` lookup/summary; обе
     outcome buttons отправляют
     `POST /admin/track-appeals/:removal_decision_id/decision` с closed
     outcome/reason/idempotency payload. Mutation требует active admin session, fresh
     auth и CSRF. Htmx имеет full-page fallback; JSON/bot endpoints,
     `/moderation/...` aliases и отдельные uphold/restore routes отсутствуют. Все
     responses — `Cache-Control: no-store`.
     Если fresh auth истекла между form render и Decision POST, request до sensitive
     reload/mutation уходит в отдельный re-auth flow, не в `422/409/503`, без outcome,
     audit/outbox или notification. Server-side сохраняется только allowlisted return
     target на canonical appeal GET с full UUID в `track_ref`; submitted
     outcome/reason/idempotency key не переносятся в URL, auth-flow или session. После
     re-auth summary/`action_state` recompute-ятся, выдаётся новый key, admin повторно
     заполняет decision.
     Successful re-auth записывает в active session только generic one-time flash kind
     `appeal_form_discarded`, без identifiers/form values/key. Canonical GET атомарно
     consume-ит его и показывает coarse localized notice о несохранённой отправке.
     Flash отсутствует в query и `409/503` catalog, исчезает после одного render, а
     terminal authoritative summary скрывает form независимо от notice.
     Expired fresh auth открывает один top-level fresh-auth flow. Full-page POST
     отвечает `303` на server-generated start URL, htmx POST — `200`/`no-store` с
     пустым body и `HX-Redirect` на тот же URL, потому что htmx не обрабатывает response
     headers на `3xx`. Оба используют один server-side `return_to` на canonical appeal
     GET; inline auth fragment отсутствует, invalid session/CSRF остаются отдельными
     fail-closed ветками.
     Appeal re-auth переиспользует обычный bot-issued `web_session` token и `/auth`
     GET/POST. Consume rotate-ит current browser session, переносит исходный
     `fresh_authenticated_at` из token record и возвращает по bound `return_to`;
     отдельного re-auth credential/table/cookie/consume endpoint нет.
     Все token-bearing outcomes общего flow используют `Cache-Control: no-store`,
     `Referrer-Policy: no-referrer` и только same-origin static assets. Caddy/app
     access/security logs пишут route без query/form body; raw token/digest отсутствуют
     в errors, analytics и traces.
     Initial token-query GET не consume-ит token и всегда отвечает `303` на clean
     `/auth`: valid branch создаёт existing auth-flow record/cookie+nonce.
     Malformed/unknown/expired/superseded branch использует stable non-secret
     `result=invalid`; target GET показывает generic-invalid без lookup/изменения
     current flow, plain `/auth` продолжает confirmation. Marker не добавляет
     state/cookie. Raw token отсутствует в body/redirect/history; JS, client storage и
     новый credential state не вводятся.
     Auth-flow record/cookie/form nonce истекают в exact original token deadline без
     sliding от confirmation GET, retry или `503`. Expired/missing component даёт
     generic-invalid page, idempotently очищает remaining flow record/cookie и не
     consume-ит, не восстанавливает и не перевыпускает source token.
     Auth-flow record содержит только source token digest, allowlisted `return_to`,
     original deadline и SHA-256 nonce hash. Independent 128-bit flow-cookie ID
     адресует Valkey record по SHA-256 cookie value; raw source token после redirect не
     хранится. Raw flow ID/nonce остаются только в `HttpOnly` cookie/hidden POST field,
     bindings читаются из token record, raw/digest values отсутствуют в telemetry/DLQ.
     Flow cookie — exact `__Host-trailbase_auth_flow` с `Secure`, `HttpOnly`,
     `SameSite=Lax`, `Path=/`, без `Domain`; expiry capped remaining source deadline.
     Success, terminal invalid/expiry и explicit cancel очищают её `Max-Age=0`;
     session cookie остаётся отдельной.
     Новый valid initial GET при existing flow-cookie атомарно удаляет previous record,
     создаёт replacement и оставляет один active browser flow. Old tab/form получает
     generic-invalid без side effects; previous source token не consume/revoke-ится и
     его link до deadline может снова заменить current flow.
     Failed flow-cookie/form-nonce validation на `POST /auth` не consume-ит source
     token и не удаляет valid flow. Missing/unknown cookie и nonce mismatch дают одну
     generic-invalid response без token/pointer/session mutations. Unknown cookie
     очищается у browser через `Max-Age=0`; valid record при wrong/stale nonce и current
     cookie сохраняются. Общий `/auth` rate limit — 10/min на IP без отдельного
     nonce-attempt counter.
     Non-transient invalid `POST /auth` использует PRG: `303` на token-free `/auth` со
     stable query marker `result=invalid`, затем generic-invalid GET без lookup current
     flow. Cleanup следует принятому error class; wrong/stale nonce сохраняет valid
     flow/cookie. Transient PostgreSQL/Valkey failure остаётся retryable `503` с
     `Retry-After` без redirect, terminal marker или credential mutations и сохраняет
     retryable flow/token/cookie.
     После successful PostgreSQL active identity/account check одна atomic Valkey
     function валидирует flow/nonce, source token и active pointer, создаёт либо
     rotate-ит browser session, consume-ит token/pointer и удаляет flow. Все credential
     mutations имеют одну linearization point; distributed PostgreSQL transaction нет.
     Два concurrent POST одной form создают ровно одну session, а проигравший terminal
     invalid и не revoke-ит успешную session.
     Link непосредственно выпускает explicit private-chat action «Подтвердить вход»;
     только этот bound validated command/callback создаёт freshness. Recent messages,
     notifications, background/browser activity, PIN и второй confirmation не
     используются.
     Re-auth доступен через любую active linked Telegram/Max identity, не только
     primary. Token bound к exact identity/provider/user/event, consume re-check-ит
     active membership; unlinked/foreign token terminal invalid, success создаёт
     account-level freshness.
     Valid same-user current browser session делает rotation credential refresh без
     `new_session` notification; token/CSRF/freshness заменяются, other sessions не
     меняются. Без такой session successful `/auth` — ordinary new login с locked-on
     security notification.
     Same-browser record сохраняет original `created_at`, обновляет
     token/CSRF/freshness/last-seen/safe-UA и one-year sliding TTL; отдельного
     `reauthenticated_at` нет.
     Actionable form получает server-generated 128-bit CSPRNG base64url
     `idempotency_key` hidden field, общий для обеих buttons и отдельный от CSRF.
     Outcome сохраняет только `idempotency_key_hash` как UNIQUE SHA-256; raw key,
     отдельная table, TTL/cleanup и logging отсутствуют. Same hash с exact normalized
     removal/outcome/reason payload возвращает committed result; mismatch даёт
     `409`/`no-store`/`idempotency_key_reused`, новый key после commit —
     `appeal_already_decided`.
     Successful commit и exact idempotent retry ведут к одному canonical GET summary с
     full track UUID. Full-page POST отвечает `303 See Other`, htmx выполняет client
     navigation к тому же GET без отдельного success fragment. Final summary
     показывает `already_decided`, committed outcome и «Решение сохранено»; refresh не
     повторяет POST. Validation остаётся на form, conflict reload-ит authoritative
     summary.
     Decision POST error matrix одинакова для htmx/full-page. `422` сохраняет form
     values/same key и показывает field errors; `409` заменяет stale form/key fresh
     summary/action_state со stable code; `503` сохраняет values/key для explicit
     manual retry и не утверждает commit result. Automatic retry/polling нет,
     `422/409` не имеют side effects, uncertain COMMIT разрешается same-key retry. Все
     responses `no-store` и не раскрывают internal errors/details.
     Closed catalog ограничивает `409` кодами `appeal_already_decided`,
     `appeal_window_closed`, `appeal_not_current`, `appeal_not_restorable` и
     `idempotency_key_reused`, а `503` — `appeal_temporarily_unavailable`.
     `409` deterministically выбирает decided/window-closed/not-current, recomputed
     restore-blocked/ready либо fresh same-case summary; `503` не утверждает
     authoritative state/commit result. Localized coarse template выбирается по
     status+code без free-form backend/internal details.
     Appeal/admin operation имеет только `uphold_removal`, не меняющий lifecycle
     state, и state-changing `restore_after_appeal`. Оба outcome ссылаются на исходный
     removal decision, требуют admin reason и создают append-only audit event без
     изменения исходного removal. Generic `resolved`, partial и custom/free-form
     outcomes отсутствуют.
     Одна permission `track_appeal_decide`, mapped только к фиксированной роли `admin`,
     защищает оба outcome; management UI требует existing fresh auth. Uphold/restore
     не разделяются на permissions, ordinary moderator endpoints и bot action
     отсутствуют. Checks выполняются до sensitive audit context, audit хранит actual
     admin actor.
     Final submit заполненной outcome+reason form является единственным confirmation
     appeal decision. Fresh-authenticated management form показывает short track ID,
     current state, выбранный outcome, его последствия и обязательную reason. Две
     визуально разделённые кнопки называются «Подтвердить удаление» и «Восстановить
     трек»; generic «Сохранить», второй modal и отдельный confirm screen отсутствуют.
     Controls disabled in-flight; backend повторно проверяет permission, fresh auth,
     single-shot и lifecycle invariants, retry остаётся idempotent.
     Form показывает static localized 10-minute fresh-auth hint без exact expiry/live
     countdown. JavaScript timer, polling, auto-refresh/submit отсутствуют; backend POST
     authoritative и направляет expiry в принятый discard/re-auth/fresh-summary flow.
     Actionable GET и Decision POST используют один server-side predicate
     `now < fresh_authenticated_at + 10 minutes`; equality уже expired. UX buffer и
     отдельный render threshold отсутствуют, а expiry во время заполнения следует тому
     же discard/re-auth flow.
     Decision POST фиксирует один server `authorization_now` в authoritative mutation
     guard перед preconditions/первой mutation. Successful check действует до конца
     transaction без второй wall-clock проверки у commit boundary, даже если physical
     commit позже deadline; последующая queue/lock latency freshness не отзывает.
     Appeal GET/POST получают `now` из единого injected application-level UTC clock с
     `java.time.Instant` semantics. Client clock и PostgreSQL `now()` freshness не
     решают; app instances требуют синхронизированных system clocks, DB timestamps не
     образуют второй freshness authority.
     Тот же application clock фиксирует `fresh_authenticated_at` при server-side
     validation/acceptance explicit bot webhook/callback. Provider event timestamp
     отбрасывается после payload validation; dedupe и exact event/identity bindings от
     него не зависят.
     Provider timestamp отсутствует в `web_session` token/session auth records; там
     остаются server freshness и opaque event/identity bindings для dedupe/consume.
     В MVP operational events/logs/metrics timestamp также не получают; observability
     использует только server-time counters по provider/validation result без raw
     payload. Delivery-delay telemetry требует отдельного будущего retention contract.
     Appeal re-auth использует общий единственный counter
     `fresh_auth_confirmation_total{provider,result}` с `provider=telegram|max` и
     `result=accepted|duplicate|invalid_event|invalid_binding|account_unavailable|internal_error`;
     identity/event/request/timestamp и дополнительные labels отсутствуют.
     `duplicate` относится только к exact provider replay после committed acceptance:
     `2xx` без нового token/link/message/session state. Pre-commit `internal_error`
     продолжает общий worker retry/DLQ и duplicate не increment-ит.
     Existing 7-day provider-event dedupe record хранит appeal fresh-auth state
     `processing|accepted`; отдельного storage нет. Ingress атомарно claim-ит processing
     и enqueue-ит event, worker после token/link issuance ставит accepted. Processing
     replay только acknowledge-ится, accepted replay increment-ит duplicate.
     Worker одной atomic Lua/Valkey operation создаёт единственный 128-bit 10-minute
     `web_session` token record и переводит processing в accepted до Bot API send.
     Crash не разделяет token/state; delivery retries используют exact same link.
     При новом confirmation той же provider identity эта issuance атомарно заменяет
     active-token pointer, удаляет previous token record и его ещё существующий raw
     delivery field. Previous provider-event marker остаётся accepted; отдельные bot
     message, revoke audit и notification не создаются.
     Previous-link `/auth` consume и new issuance линейризуются atomic Valkey operations
     над pointer. Consume-first удаляет matching token record/pointer и может завершить
     auth; issuance-first заменяет pointer и делает old link terminal invalid. Поздняя
     issuance не отзывает создаваемую/созданную browser session; distributed
     Valkey/PostgreSQL transaction отсутствует.
     Active-token pointer — отдельный non-secret Valkey key по internal identity UUID,
     содержащий только SHA-256 token digest, с `PEXPIREAT` exact token deadline. Atomic
     consume удаляет его с token record, replacement заменяет; missing/expired pointer
     или target terminal invalid и idempotently очищается. Delivery data, sliding TTL
     и janitor отсутствуют.
     `web_session` token record адресуется только по `SHA-256(raw_token)`, тому же
     digest, что pointer, без raw token в record/key. 128-bit CSPRNG entropy позволяет
     unsalted SHA-256 без custom HMAC/salt. Raw token остаётся лишь в link/browser
     request и short-lived delivery field; raw/digest запрещены в
     logs/metrics/traces/DLQ.
     Raw link token временно хранится только в delivery field accepted dedupe record до
     successful send/token expiry. После очистки остаётся seven-day non-secret marker;
     secret отсутствует в logs/metrics/traces/DLQ и отдельном namespace.
     Valkey 9.x minimum позволяет atomic issuance задать `HPEXPIREAT` на token deadline;
     successful send делает idempotent `HDEL`. Accepted key TTL остаётся seven days без
     janitor или отдельного expiry key.
     Если appeal re-auth link не доставлен до token/delivery-field expiry, worker
     завершает delivery terminally без stale send, re-issuance, изменения accepted
     marker или отдельного user notification. Exact replay остаётся `duplicate`; новый
     provider event/freshness/token/link создаёт только новое explicit «Подтвердить
     вход» action. Failure отражается только общими Bot API delivery metrics/alert без
     raw token.
     Post-issuance delivery выполняет максимум пять total attempts с exponential
     backoff/jitter. Retry разрешён только для timeout/network, `429` и `5xx`, если
     следующая attempt с `Retry-After` помещается в original token deadline. Остальные
     `4xx`, exhausted budget или retry за deadline terminal сразу с idempotent `HDEL`,
     сохранением accepted marker и без DLQ/late replay. Pre-issuance `internal_error`
     использует общий worker retry/DLQ flow.
     Appeal outcome переиспользует account-reactivation reason contract:
     `reason_code text NOT NULL` с CHECK на
     `support_request_verified|administrative_correction|other`, без PostgreSQL enum.
     Optional `reason_note` обязательна для `other`, после trim содержит 1–1000
     Unicode code points без control/newline. Code/note только в append-only admin
     audit, не в owner notification/logs; отдельного appeal catalog нет.
     Appeal outcome single-shot: `removal_decision_id` имеет UNIQUE constraint, first
     commit wins. Same-key retry того же outcome возвращает сохранённый result без
     второго audit/outbox; другой или competing outcome получает
     `409`/`no-store`/`appeal_already_decided` без side effects. Reopen/second appeal
     отсутствуют, correction оформляется отдельной append-only lifecycle operation.
     `restore_after_appeal` возвращает прежний immutable approved
     `current_revision_id` в published без новой ordinary moderation. Если approved
     revision до removal не было, track становится private/editable и требует явного
     owner resubmit с новой immutable revision. Removed pending revision остаётся
     decided, её export — `discarded`, и она никогда повторно не входит в queue.
     Moderator removal сохраняет private sanitized export последнего approved
     `current_revision_id` до `uphold_removal` либо 90-day purge. Canonical routes
     отвечают `404`, новые presigned URL не выдаются, object доступен только backend.
     `restore_after_appeal` атомарно возвращает snapshot в published и пишет
     audit/outbox без regeneration job или `restoring` state; uphold/purge удаляют
     object. Unapproved pending export остаётся immediate `discarded`+delete.
     Uphold сохраняет original `purge_at`: export удаляется сразу, остальные retained
     data — в исходный 90-day deadline. Restore атомарно ставит `purge_at = NULL`;
     extension/new timer нет, error/conflict/retry срок не меняют. Outcome и purge
     worker сериализуются track lock: purge-first делает restore invalid, restore-first
     заставляет worker no-op после re-check.
     Первый новый appeal outcome под track lock требует current active removal,
     отсутствующий outcome и `now() < purge_at`. Expiration закрывает uphold и restore
     даже при lag purge worker-а; support request окно не продлевает, newer lifecycle
     state закрывает обе операции. До deadline full-track lock и unavailable retained
     export блокируют только restore, audit-only uphold остаётся доступным. Committed
     outcome по-прежнему обслуживает same-outcome idempotent retry.
     Restore под track lock требует current `moderator_removed`, matching active
     `removal_decision_id`, unexpired purge deadline, отсутствие newer lifecycle event
     и full-track lock, плюс ready retained export для published branch. Иначе
     `409`/`no-store`/`appeal_not_restorable` не создаёт outcome, audit/outbox или
     delete; temporary block можно устранить и retry-нуть до deadline. Full lock не
     блокирует audit-only uphold.
     Первый committed appeal outcome атомарно создаёт locked-on owner web-inbox record
     и primary-bot outbox. Uphold содержит confirmation fact, short track ID и purge
     deadline; restore — canonical link для published либо edit/resubmit для private.
     Admin reason/note, actor, audit ID/context скрыты. Error/conflict/idempotent retry
     notification не создают; disabled owner следует moderation suppression без
     backlog/replay.
     Отдельного track lifecycle state `rejected` нет.
     `changes_requested` хранит один `reason_code text NOT NULL` с DB CHECK на
     `metadata_correction`, `geometry_correction`, `privacy_correction`,
     `classification_correction`, `license_or_attribution`, `other`, без enum/array.
     Optional owner-visible `reason_note` обязателен для `other`, после trim содержит
     1–1000 Unicode code points без control/newline и не логируется; localized label
     и note входят в locked-on notification. Removal/issue code namespaces отдельны.
     First-publication review показывает полный snapshot. Review новой revision уже
     published track по умолчанию показывает field diff и old/new geometry overlay
     относительно nullable `base_revision_id`, который равен `NULL` для first
     publication и закрепляет same-track current revision при создании update.
     Composite FK защищает lineage, self-reference запрещён. Approval под track lock
     требует совпадения baseline с `tracks.current_revision_id`; mismatch остаётся
     stale без mutation/audit или automatic rebase. Полный pending snapshot доступен
     отдельно, а moderation decision применяется ко всей immutable revision без
     partial approval. Авторский resubmit после `changes_requested` дополнительно
     хранит обязательный same-track `correction_of_revision_id` на непосредственно
     возвращённую immutable revision и сохраняет её `base_revision_id`; для прочих
     revisions correction pointer равен `NULL`. Его default review показывает
     correction diff с сохранёнными code/note, а total diff от baseline и полный
     snapshot остаются вторичными views.
     До approval private sanitized object доступен только backend workers:
     owner/moderator UI не получает presigned URL и не имеет отдельного authenticated
     download route; download появляется только после publication через canonical
     route.
     Changes-request или moderator removal pending revision ставит
     `export_state = discarded` и high-priority delete; новый revision после
     changes-request export не переиспользует. Late worker completion не меняет
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
     отвечает одинаковым `404`; otherwise published full-locked track получает
     generic `503` без `Retry-After`, metadata или integrity details. Raw не
     публикуется, bot не раскрывает signed URL.
     Все canonical `302/404/429/5xx` имеют `Cache-Control: no-store`, без CDN
     cache/`ETag`; каждый request заново проверяет limit/publication state.
     Active full lock прекращает выдачу новых presigned URLs, но не удаляет/ротирует
     object; уже выданная ссылка живёт не более исходного five-minute TTL.
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
     export с retry/DLQ/alert, кроме private retention последнего approved export после
     moderator removal до uphold/90-day purge. Выданный URL после delete не работает,
     а при задержке ограничен исходным five-minute TTL. Download proxy не вводится.
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
