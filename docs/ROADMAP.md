# TrailBase — Roadmap (vertical slices)

Декомпозиция дизайна (см. `architecture.md` + `docs/adr/`) на независимо доставляемые вертикальные срезы (tracer bullets). Каждый срез — **runnable end-to-end**, observable через конкретное поведение, а не горизонтальный слой.

Точные security, data-model, runtime и API-инварианты находятся в
[Implementation Contract](IMPLEMENTATION-CONTRACT.md). Он уточняет этот roadmap и
имеет приоритет над старыми детальными формулировками ниже.

M01 — prerequisite всех остальных. M02 → M03 → M04 — backbone. M05/M06/M07/M08 — independent extensions поверх backbone (могут разрабатываться параллельно после M04).

```
M01 Foundation
   │
   ▼
M02 Identity ──▶ M03 Upload ──▶ M04 Catalog Render
                              │
                ┌─────────────┼─────────────┐
                ▼             ▼             ▼             ▼
             M05 POI        M06 Search    M07 Class      M08 Photo/Elev
```

---

## M01 — Foundation

**Объём**: проект, БД-схема core-таблиц, HTTP-skeleton, базовый render.

**Файлы/компоненты**:
- `deps.edn` (Clojure deps + aliases: dev/test/migrate).
- `bb.edn` tasks: `migrate`, `rollback`, `run-dev`, `repl`.
- `resources/migrations/` (migratus): `001-init.up.sql/down.sql` — core-схема.
- `src/trailbase/core.clj` — Ring/reitit app skeleton, `/health` endpoint.
- `src/trailbase/db.clj` — next.jdbc datasource + hugsql loader.
- `src/trailbase/render.clj` — Hiccup → HTML, базовый layout + partials.

**Core data model (M01 закладывает основу; M02/M03/M05 дополняют)**:
- `users`, `user_identities`, append-only `user_agreements`
- `tracks` как stable identity и immutable `track_revisions` для versioned content
- `tags`, а revision-tag связи добавляются в M07
- `locations` как stable identity и `location_revisions`, annotations добавляются в M05
- Valkey для browser sessions, web-session/link tokens, ephemeral chat interaction
  state, rate limits и async streams

**Acceptance**:
- `bb migrate` создаёт PostGIS-расширение и все core-таблицы.
- `bb run-dev` поднимает HTTP-сервер; `/health/live` и внутренний `/health/ready`
  выполняют разные проверки.
- `GET /` рендерит Hiccup-страницу с layoutом; partial добавляется через `hx-get`.
- next.jdbc datasource доступен в REPL; hugsql queries загружаются.
- Перед этим Docker Compose поднимает PostgreSQL/PostGIS, MinIO и Valkey одной командой.

**Non-goal**: auth, реальные треки, бот.

---

## M02 — Messenger Identity + Optional Web Session

**Объём**: Telegram/Max webhook identity → account и chat operations без обязательного
web activation; optional deep-link → безопасный GET/POST flow → Valkey browser session;
explicit identity linking, roles и active-session management.

**Файлы/компоненты**:
- `src/trailbase/bot/telegram.clj` — raw Bot API через hato (setWebhook/sendMessage);
  polling не реализуется.
- `src/trailbase/bot/max.clj` — raw Bot API (Max/NetM), аналогично Telegram.
- `src/trailbase/bot/core.clj` — общая проверка messenger identity, command routing и
  обработка обычного `/start`, `/start <link-token>` и optional browser-session
  deep-link.
- Plain `/start` атомарно создаёт account/identity и минимальный agreement record,
  затем показывает описание TrailBase/bot, главное меню, ссылки на актуальные rules и
  CC BY 4.0 и уведомление, что дальнейшее использование означает автоматическое
  согласие, включая будущие изменения правил. Кнопки «Согласен» и pending state нет;
  linking payload использует отдельную fail-closed ветку.
- Главное меню обоих providers содержит «Поиск», «Загрузить GPX», «Мои
  треки/черновики», «Настройки» и «Помощь» в одном порядке. «Правила» и «Лицензия»
  остаются вторичными ссылками вне главного меню.
- «Настройки» содержит «Профиль» (public display name/language), «Мессенджеры»
  (linked identities/primary provider), «Уведомления», «Сессии» (active web sessions,
  revoke/logout-all) и «Аккаунт» (деактивация). Email/phone/password/recovery
  settings отсутствуют.
- Все settings operations доступны в private chat без web session. Profile и
  notifications меняются в chat; provider linking завершается через target
  `/start <link-token>`; sessions доступны для list/revoke/logout-all; deactivation
  выполняется после fresh auth и confirmation. Web UI — optional mirror.
- Public `users.display_name` меняется сразу без fresh auth/pre-moderation и
  audit-ится. После NFC/trim значение должно быть непустым, без control/newline
  characters и не длиннее 200 Unicode code points; abuse идёт через общую moderation
  policy.
- Attribution на public track pages всегда использует текущее `users.display_name`,
  включая старые published tracks после реактивации и смены имени; отдельного
  revision-level author-name snapshot для web нет. Уже созданный GPX export остаётся
  immutable со старым именем, а новый export использует имя на момент генерации.
- UI locale выбирается только из `ru`/`en` и применяется к bot и web UI.
  Поддерживаемый provider language служит initial hint, но не перезаписывает ручной
  выбор. `simple` остаётся техническим content fallback, не user locale.
- Security notifications и action-required moderation (`changes_requested`,
  `rejected`) locked-on и всегда доставляются в web inbox/primary bot. Catalog,
  informational и остальные moderation results являются configurable preferences.
- Пока account deactivated, доставляются только locked-on security/account-lifecycle
  notifications через web inbox и primary linked messenger. Все moderation, catalog
  и informational notifications подавляются без backlog/replay; domain events и
  audit сохраняются.
- Reactivation notification показывает только факт/time, `TRAILBASE_SUPPORT_URL` и
  инструкцию снова открыть bot. Admin identity, internal reason, audit ID и tokens не
  раскрываются; reason остаётся только в audit.
- Deactivation transaction отменяет `pending` non-security notification outbox
  delivery intents и suppress-ит связанные unread web-inbox records с обычным
  retention. Уже claimed `sending` delivery может завершиться;
  security/account-lifecycle records не отменяются.
- New-account defaults: optional moderation results, включая `published`, enabled;
  catalog и informational disabled. Locked-on categories не имеют configurable
  default.
- Один account-level optional toggle применяется одновременно к web inbox и primary
  bot; per-channel preferences отсутствуют. Disabled category не создаёт notification
  и bot delivery, но не подавляет domain state/event/audit. Для active account
  locked-on delivery неизменна.
- Preference snapshot применяется в transaction создания notification/outbox intent.
  Позднее изменение влияет только на будущие events и не отменяет queued delivery;
  dispatcher не перечитывает settings.
- «Мои треки/черновики» показывает в chat единый список owned tracks и
  upload/draft flows со статусами `draft`, `processing`, `pending_review`,
  `changes_requested`, `published`, `rejected`. Требующие действия пользователя
  элементы идут первыми, остальные — по последнему изменению; web session не нужна.
- Archived tracks скрыты из default list и доступны через вторичный filter «Архив».
  Карточка показывает оставшийся срок до 30-дневного purge и действие
  «Восстановить»; early permanent delete в MVP отсутствует.
- Дополнительных status filters в MVP нет: list views ограничены default «Все» и
  «Архив». Требующие действия entries остаются в начале «Все».
- User archive сохраняет pre-archive status/current revision. Restore до `purge_at`
  атомарно возвращает то же состояние: прежний approved snapshot публикуется без
  повторной moderation, private track остаётся private. Moderator removal не
  восстанавливается этим flow.
- «Восстановить» не требует второго confirmation: valid owner callback атомарно
  проверяет `archived`/`purge_at` и выполняет idempotent restore. Foreign, stale или
  expired callback state не меняет.
- Карточка показывает только допустимые для текущего статуса действия:
  редактирование для `draft`/`changes_requested`, прогресс/cancel для `processing`,
  read-only просмотр для `pending_review`, просмотр/download/new revision/archive
  для `published`, причину и новую исправленную revision для `rejected`.
  Delete/archive требуют отдельного подтверждения.
- Список показывает 10 элементов на страницу с controls «Назад»/«Далее» и
  deterministic server-side keyset cursor. Provider callback содержит только opaque
  ID; cursor и provider/chat/message/requester binding хранятся в Valkey.
- List controls истекают через 15 минут от открытия без продления. Expired callback
  или потеря Valkey state не редактирует старое сообщение и предлагает заново открыть
  «Мои треки/черновики».
- «Правила»/«Лицензия» возвращают обычные HTTPS-ссылки; полные документы и их chat
  copies не хранятся. Rules link всегда открывает актуальный текст.
- Автоматическое agreement первого plain `/start` служит standing consent для CC BY
  4.0 contributions. `user_agreements` хранит только `user_id`, `accepted_at`,
  `source = /start`, notice SHA-256 и rules/license URLs; unique `user_id` исключает
  повторную запись. IP, raw tokens и callback payload не сохраняются.
- Повторный plain `/start` не изменяет `accepted_at`, notice hash или URLs; он только
  снова показывает notice/menu и обновляет provider profile snapshot.
- Деактивация и административная реактивация account сохраняют исходный
  `user_agreements` record неизменным и не требуют нового agreement. Физическое
  удаление account остаётся отдельной data-retention/privacy policy вне MVP.
- Published tracks деактивированного account остаются в public catalog с прежним
  TrailBase author display name и CC BY 4.0 attribution. Деактивация блокирует новые
  login/mutations и сохраняет private content private; скрытие или удаление track
  остаётся отдельной lifecycle operation.
- Self-deactivation после fresh auth требует только explicit confirmation: reason
  code/free text/feedback не запрашиваются. Audit сохраняет actor=user, action,
  interface/provider, request ID и timestamp.
- Final confirmation без dynamic counts перечисляет session/token revocation,
  блокировку account-specific operations, сохранение public CC BY 4.0 tracks/private
  content, private-only parse completion, removal pending review из queue и
  admin-only reactivation через `TRAILBASE_SUPPORT_URL`. Actions:
  «Деактивировать аккаунт»/«Отмена».
- Confirmation expires exactly at `fresh_authenticated_at + 10 minutes` без sliding.
  Chat использует 128-bit opaque ID с server-side
  user/provider/chat/message/requester/purpose binding; web — active session, CSRF и
  user/purpose binding. Confirm/Cancel single-use; invalid/stale/replay ничего не
  меняет и требует нового fresh auth.
- `sensitive_operation_confirmations` находится в PostgreSQL и хранит только SHA-256
  opaque ID, bindings/timestamps и `consumed_at`; raw ID остаётся в control/form.
  Confirm под row lock атомарно consume-ит state, деактивирует account и пишет
  audit/outbox; Cancel только consume-ит state. Cleanup consumed/expired rows — 24
  часа.
- PostgreSQL active status проверяется при каждой session validation и каждом
  auth/link token consume, немедленно закрывая доступ после deactivation commit.
  Confirm transaction пишет idempotent cleanup command без raw tokens; worker удаляет
  account Valkey keys с retry/DLQ. Failure не открывает доступ, backlog/DLQ alert-ит
  operator.
- Дополнительный credential epoch/version отсутствует: deactivation flow удаляет
  account sessions и outstanding auth/link tokens непосредственно из Valkey через
  этот cleanup command.
- Если PostgreSQL недоступен, session/token authorization работает fail-closed без
  cached-active fallback: account-specific HTTP flow возвращает `503` с
  `Retry-After` без session/mutation, а bot event проходит обычный transient
  retry/DLQ без domain changes. Valkey credential сам по себе доступ не даёт.
- Session-validation `503` содержит `Cache-Control: no-store`, не очищает cookie или
  Valkey session и не продлевает sliding TTL/`last_seen_at`. После восстановления
  PostgreSQL ещё не expired session снова пригодна; очистка cookie и revoke
  выполняются только для подтверждённого invalid, expired или disabled состояния.
- Public catalog не показывает badge, причину или иной marker деактивации автора.
  Disabled status видят только сам пользователь через нейтральный auth response и
  администраторы через management/audit.
- In-flight parse деактивированного owner может завершиться только private draft.
  `pending_review` сохраняет status, но исключается из moderator queue; publish
  fail-closed re-check-ит active owner. Реактивация возвращает pending item в очередь
  без resubmit, а draft пользователь отправляет сам. Новый `suspended` status не
  вводится.
- Deactivation и approve/publish используют owner row lock и единый порядок
  `user -> track -> revision`. Первый commit определяет результат: approval-first
  оставляет опубликованный track public после деактивации, deactivation-first не
  допускает публикацию.
- Private chat disabled identity не создаёт новый account. Ему доступны только
  `/start` с нейтральной инструкцией, help/rules/license и read-only public search под
  stateless principal; upload, owned data/settings и auth/link tokens запрещены.
- Встроенного reactivation-request flow нет. Disabled `/start` показывает обязательный
  production `TRAILBASE_SUPPORT_URL`; admin после out-of-band проверки реактивирует
  account в management UI с fresh auth, обязательной причиной и audit. Bot не
  создаёт ticket и не пересылает free-form сообщения.
- Reactivation reason code обязателен:
  `support_request_verified`, `administrative_correction` или `other`. Audit note
  optional, но обязательна для `other`; после trim она должна быть непустой, без
  control/newline characters и не длиннее 1 000 Unicode code points. Code/note
  остаются только в append-only audit и не логируются.
- Reactivation только возвращает account в active: linked identities, roles,
  settings, private content и pending moderation снова доступны. Revoked sessions,
  invalidated auth/link tokens и suppressed notifications не восстанавливаются;
  agreement не меняется, а новый web login инициируется отдельно через bot.
- Изменение правил по ссылке не создаёт agreement records, notifications или domain
  state, не ограничивает account/session capabilities и не требует повторного
  consent.
- Agreement является account-level состоянием по `user_id`. После explicit linking
  новая messenger identity сразу использует capabilities target account; отдельного
  notice/agreement record нет, исходное evidence первого `/start` не меняется.
- `src/trailbase/auth.clj` — token.issue/verify, session cookie set/get, identity bind/link.
- Web-session/link tokens и sessions находятся в Valkey; PostgreSQL хранит users,
  messenger identities, roles и audit.
- `GET /auth` показывает confirmation; `POST /auth` атомарно расходует token и выдаёт
  `__Host-trailbase_session`, но не активирует account.

**Acceptance**:
- Первый валидированный Telegram `/start` в private one-to-one chat для неизвестной
  identity атомарно создаёт active account и identity; повторные или конкурентные
  deliveries идемпотентно resolve в тот же account.
- Первый plain `/start` атомарно создаёт account, identity и один agreement record;
  повторный или конкурентный delivery не создаёт дубликат.
- Повторный `/start` оставляет agreement record byte-for-byte неизменным, но может
  обновить provider profile snapshot и заново показать notice/menu.
- После деактивации и административной реактивации исходный agreement record остаётся
  неизменным; повторное agreement не требуется.
- Деактивация не скрывает и не удаляет published tracks: они остаются public с
  прежним author display name и CC BY 4.0 attribution; private content не становится
  публичным, а track removal выполняется отдельной lifecycle operation.
- Self-deactivation UI не содержит reason или feedback field; audit record не
  содержит пользовательский free text.
- Confirmation показывает все принятые последствия и exact primary/secondary
  actions, но не вычисляет counts, способные устареть до mutation.
- Confirmation callback после original fresh-auth deadline, с неверным binding или
  после первого Confirm/Cancel не меняет account. Web mutation без valid CSRF/state
  также fail-closed.
- Concurrent/replayed Confirm не может дважды выполнить domain mutation: PostgreSQL
  row lock и `consumed_at` дают единственный winner; confirmation rows старше 24
  часов после terminal/expiry удаляются janitor.
- Даже до физического Valkey cleanup session/token deactivated account не
  авторизуется. Cleanup command повторяем и не содержит credentials; DLQ не меняет
  committed account state.
- При недоступном PostgreSQL Valkey session или token не авторизует operation:
  HTTP получает `503` с `Retry-After` без session/mutation, bot event повторяется как
  transient failure и при исчерпании попыток попадает в DLQ без domain changes.
- Session-validation `503` не выглядит как logout: cookie и Valkey session
  сохраняются, `last_seen_at`/sliding TTL не обновляются, ответ имеет
  `Cache-Control: no-store`, а после восстановления PostgreSQL та же неистёкшая
  session снова работает.
- Public response для такого track не раскрывает деактивацию автора; account status
  доступен только самому пользователю и администраторам.
- Деактивация не допускает новую публикацию: parse может завершить private draft,
  `pending_review` временно отсутствует в moderator queue, а approve/publish
  отклоняется до реактивации.
- Concurrent deactivation/approval детерминирован row lock: только approval,
  committed до deactivation, может опубликовать revision; обратный порядок не меняет
  public state.
- Disabled identity в private chat resolve-ится в прежний account, видит ограниченное
  static/read-only меню и не получает account-specific данные или mutation controls.
- Production без valid absolute HTTPS `TRAILBASE_SUPPORT_URL` не стартует. Disabled
  `/start` показывает эту ссылку; единственная reactivation mutation остаётся в admin
  management UI и создаёт locked-on lifecycle notification.
- Reactivation с unknown/missing reason code, missing `other` note или note длиннее
  1 000 Unicode code points отклоняется без изменения account; accepted code/note
  находятся только в audit.
- После reactivation следующий validated private-chat event показывает обычное меню
  и сохранённые account capabilities. Ни одна прежняя web session/token/notification
  не воскресает; lifecycle notification не содержит auth token.
- Reactivation notification не раскрывает admin identity/reason/audit ID и содержит
  только timestamp, support URL и безопасную инструкцию вернуться в bot.
- После реактивации смена `users.display_name` сразу меняет attribution всех старых
  public track pages. Ранее сгенерированный GPX не переписывается; новый export
  содержит имя, актуальное на момент генерации.
- Ответ на plain `/start` показывает информацию о боте, главное меню, rules/CC BY 4.0
  links и явно сообщает, что использование бота означает автоматическое согласие.
  Кнопки «Согласен» и отдельного consent callback нет.
- Главное меню Telegram и Max показывает ровно пять основных действий: «Поиск»,
  «Загрузить GPX», «Мои треки/черновики», «Настройки» и «Помощь»; правила и лицензия
  не добавляют отдельные пункты в это меню.
- «Настройки» показывает ровно пять разделов: «Профиль», «Мессенджеры»,
  «Уведомления», «Сессии», «Аккаунт»; recovery credentials/channels не предлагаются.
- Каждый раздел «Настроек» выполняет разрешённые операции без web session в private
  chat; sensitive operations сохраняют fresh-auth/confirmation guards, а web UI не
  является обязательным переходом.
- Valid display name применяется сразу и не перезаписывается provider snapshot;
  invalid/empty/over-200 input отклоняется, каждое изменение audit-ится.
- Ручной выбор `ru`/`en` одинаково действует в Telegram/Max и web и сохраняется при
  последующих provider events; `simple` отсутствует в locale selector.
- «Уведомления» не позволяет отключить security или action-required moderation, но
  позволяет изменить catalog/informational/other moderation preferences.
- Новый account сразу показывает moderation-result notifications как enabled, а
  catalog/informational как disabled.
- Изменение optional toggle одинаково управляет будущими web-inbox и primary-bot
  notifications; отдельных channel toggles UI не показывает.
- Уже созданная notification/outbox delivery завершается по transaction-time
  preference snapshot, даже если пользователь затем изменил toggle; deactivation
  suppression является отдельным lifecycle exception.
- Для deactivated account создаются только security/account-lifecycle notifications;
  moderation/catalog/informational delivery отсутствует и не воспроизводится после
  реактивации, но соответствующие domain events/audit не теряются.
- Pending non-security delivery при деактивации становится cancelled, связанный unread
  inbox record — suppressed и невидимым после реактивации. Уже `sending` delivery
  разрешено завершиться; отзыва отправленного сообщения нет.
- Chat session card показывает safe device/browser summary, `created_at`,
  `last_seen_at`, current expiry и «Завершить». IP/geolocation/full User-Agent/session
  ID/token отсутствуют; target хранится только в bound server-side control state.
  Метки «текущая» в chat нет.
- Revoke одной session требует short confirmation с тем же summary без fresh auth.
  Confirm атомарно re-check-ит owner/target; already-ended result идемпотентен.
  `logout-all` остаётся отдельной fresh-auth operation.
- Chat `logout-all` после fresh auth/confirmation атомарно завершает все active web
  sessions account и сообщает actual revoked count. Messenger identities, chat
  access и account state не меняются.
- «Мои треки/черновики» без web session возвращает единый список owned entries,
  показывает их нормализованный статус и сортирует требующие действия пользователя
  раньше остальных.
- Default list не содержит archived tracks; filter «Архив» показывает срок до purge и
  позволяет восстановление в пределах 30 дней, но не досрочный permanent delete.
- UI не показывает дополнительных status filters кроме «Все» и «Архив».
- Restore user-archived track сохраняет `track_id`, не создаёт новую revision и
  возвращает pre-archive status/current revision; moderator-removed track через
  «Архив» восстановить нельзя.
- Одно valid нажатие «Восстановить» завершает restore без дополнительного prompt;
  повторная доставка идемпотентна, а invalid/expired callback не меняет track.
- Карточка owned entry показывает status-dependent actions; `pending_review` остаётся
  read-only, а delete/archive mutation невозможна без отдельного confirmation.
- Переходы «Назад»/«Далее» показывают не более 10 entries, используют keyset
  pagination и не принимают callback с несовпадающим
  provider/chat/message/requester binding.
- Перелистывание не продлевает исходный 15-минутный TTL; после expiry старое
  list message остаётся неизменным, а bot предлагает открыть новый список.
- В Telegram/Max «Правила»/«Лицензия» являются обычными HTTPS-ссылками; rules URL
  показывает актуальный текст без version-specific account behavior.
- Durable agreement record фиксирует notice hash, показанные URLs, `accepted_at` и
  `/start` source без ephemeral callback state.
- Любое изменение текста правил не требует нового agreement, не создаёт notification
  и не блокирует существующие browser или chat capabilities.
- Созданная Telegram identity может выполнять разрешённые chat commands без web
  session.
- Telegram bot `/start` может предложить optional browser deep-link; confirmation POST
  создаёт browser session, но не является условием account/chat access.
- `/start <link-token>` от второго provider атомарно привязывает candidate identity к
  target account и не создаёт отдельный account; browser completion не требуется.
- Привязанная identity не создаёт вторую запись `user_agreements`; agreement и
  capabilities принадлежат target `user_id`.
- Identity, уже принадлежащая другому account, не переносится; автоматического merge
  accounts нет.
- Невалидный, просроченный, использованный или provider-mismatched link token не
  создаёт account и не fallback-ит на обычный `/start`; plain `/start` без payload
  остаётся отдельным явным созданием account.
- В group/channel contexts `/start`, выпуск tokens и linking не создают и не изменяют
  account/identity state; bot без auth token направляет пользователя в private chat.
- Max bot поддерживает тот же identity/chat contract и optional `/auth` endpoint,
  identity `max:*`.
- Session cookie проверяется middleware; запрос на web UI без cookie предлагает вход
  через Telegram/Max.
- Logout текущей сессии, logout-all и active sessions management работают в web UI и
  private chat; sessions имеют sliding TTL один год.
- Session list в chat не раскрывает IP или identifiers и позволяет выбрать конкретную
  session по безопасному summary через opaque bound control.
- Одна session не revoke-ится первым нажатием: confirmation требуется, fresh auth —
  нет; stale/already-ended target не затрагивает другие sessions.
- Подтверждённый `logout-all` завершает все web sessions, существующие в момент
  mutation, но не unlink-ит messenger identity и не деактивирует account.
- Email, phone, password и recovery flow отсутствуют.

**Non-goal**: фактическая загрузка и поиск треков — M03 и M06.

---

## M03 — GPX Upload + Parse

**Объём**: web/bot upload → exact original private raw в S3 → async parse job → permanent
private draft → moderated immutable revision.

**Файлы/компоненты**:
- `src/trailbase/storage/s3.clj` — aws-simple-sign подпись + hato put/get-object + presigned URL.
- `src/trailbase/parse/gpx.clj` — hardened pull-parser → 2D MultiLineString,
  elevation profile, duration variants, metrics и warnings.
- `src/trailbase/tracks.clj` — upload flow (write S3 + insert DB), simplify-geometry (compute z11/13/15 в PostGIS-side helper или Clojure-side `simplify`).
- M03 migration добавляет `raw_objects` как durable owner/storage entity и
  `raw_object_id` foreign keys в `upload_jobs` и `track_revisions`; snapshots не
  дублируют S3 storage fields.
- `track_revisions.raw_object_id` для GPX-derived snapshots использует
  `ON DELETE RESTRICT`; nullable `upload_jobs.raw_object_id` — `ON DELETE SET NULL`.
  Job FK existence само по себе не является retention pin.
- Та же migration добавляет partial covering quota index по
  `raw_objects(owner_id, state)` для `pending|ready` с included
  `byte_size`; counter columns в `users` отсутствуют.
- PostGIS-функция в `003-tracks-geom.up.sql` (migratus) —
  `populate_simplified_geometries(track_revision_id)` заполняет три revision columns.
- Activity-auto-suggestion: рудиментарный classifier показывает до трёх ranked
  вариантов с confidence; пользователь обязан выбрать activity явно.
- Canonical duration default выбирается как GPX moving, затем GPX elapsed, затем
  unknown. Summary показывает оба рассчитанных значения и выбранный source; manual
  override или explicit unknown не удаляет auto-derived values.
- Manual duration — целые секунды `1..31_536_000`; unknown хранится как `NULL`, а не
  zero. Outlier comparison использует moving, иначе elapsed; симметричный ratio
  строго больше 10 требует warning confirmation, но остаётся допустимым. Без auto
  candidate warning отсутствует.
- Durable outlier acknowledgement в PostgreSQL привязано к manual value, comparison
  source/value и algorithm version. Mutation проверяет lease generation; resume и
  takeover сохраняют acknowledgement, а duration change/reparse/version change
  инвалидируют. Valkey содержит только prompt/control token.
- Track moderation summary показывает acknowledged duration outlier: manual value,
  comparison auto value/source и ratio. Flag не вызывает automatic reject и не
  меняет queue priority.
- Moderator не редактирует pending track revision: approve as-is либо
  `changes_requested` с причиной. Авторская правка создаёт новый immutable revision,
  повторно запускает validation/outlier acknowledgement и может переиспользовать raw
  GPX; administrative repair остаётся отдельной audited operation.
- Raw GPX сохраняется без application encryption или преобразования: private S3
  object содержит exact original upload bytes. Data/master keys, crypto envelope,
  keyring, rewrap и decrypt path для raw отсутствуют.
- Raw доступен только backend workers для parse/re-parse. Public и owner-only
  download routes, presigned raw URL и HTTP streaming endpoint в MVP отсутствуют;
  пользователь скачивает только опубликованный sanitized GPX.
- После успешного parse raw хранится без отдельного TTL до физической очистки
  связанного draft/track, включая published и archive/appeal retention, и всё время
  учитывается в user raw-storage quota. Подтверждённое удаление draft или purge track
  удаляет raw; 24-hour janitor касается только incomplete/orphan objects.
- Каждый новый upload создаёт новый immutable exact raw object без content-based,
  cross-track или cross-user dedup. Re-parse/metadata revision того же track без
  нового upload переиспользует durable reference на существующий object, не копирует
  source bytes и не увеличивает quota; object удаляется после последней reference.
- `raw_objects` row владеет `owner_id`, opaque S3 key, exact `byte_size`, private
  HMAC, lifecycle state и timestamps.
  `upload_jobs` и `track_revisions` хранят только `raw_object_id`; quota и cleanup
  работают по этой durable entity.
- Retained revision всегда pin-ит raw. Upload job pin-ит его только во время
  продолжимого upload/parse либо explicit transient retry до 24-hour incomplete
  deadline. Successful job передаёт pin revision; cancel/permanent failure снимает
  его немедленно. Историческая terminal job row не блокирует cleanup.
- Reuse и last-reference cleanup берут locks `users -> raw_objects FOR UPDATE`.
  Reuse проверяет owner, same track lineage и `ready`; cleanup под lock повторяет pin
  query перед `delete_pending`+outbox. State не оживляется: cleanup-first означает
  reject позднего reuse и новый upload.
- Lifecycle raw row ограничен `pending`, `ready`, `delete_pending`: row создаётся до
  PUT, revision принимает только checksum-validated `ready`, а потеря последней
  reference либо 24-hour incomplete cleanup атомарно ставит `delete_pending` и
  cleanup outbox command. После успешного S3 delete row удаляется; upload errors
  остаются в `upload_jobs`, delete retries/DLQ — в cleanup command.
- Quota contribution: новый `pending` атомарно резервирует 10 MiB независимо от
  provider/HTTP size metadata; `ready` считает actual byte size;
  `delete_pending` — zero с commit transition. Terminal upload failure ставит
  `delete_pending` сразу, 24-hour janitor остаётся fallback. Cleanup lag/DLQ не
  расходует user quota.
- Raw cleanup outbox payload содержит только `raw_object_id`. Worker загружает
  `delete_pending` row, берёт opaque S3 key, считает DELETE success/`404` успехом и
  после этого удаляет row. Missing row при replay — success; wrong state или
  retained revision fail-closed проходят retry/DLQ с alert. Crash между S3 и DB
  delete закрывается тем же idempotent replay.
- Quota-changing transaction берёт owner `users` row `FOR UPDATE`, выполняет indexed
  SQL sum contributions из `raw_objects` и вставляет `pending` в той же transaction.
  `users.raw_bytes_*` counters и reconciliation отсутствуют.
- Private sanitized export pre-generate-ится для immutable pending revision.
  Approve доступен только при durable `export_state = ready` и в одной transaction
  переключает revision/track в published/current. Export failure оставляет
  `pending_review`, не меняет старый current и проходит retry/DLQ; public
  `publishing` status отсутствует.
- До approval private sanitized object доступен только backend workers. Owner и
  moderator работают через draft/review UI без presigned URL или отдельного
  authenticated download route; moderator видит `export_state` и retry action.
  Download появляется только после publication через canonical route.
- Moderator queue показывает pending/error export badge и оставляет доступными review,
  `changes_requested` и reject; Approve disabled до ready. После automatic retry
  exhaustion moderator может идемпотентно requeue export, operator alert-ится, а
  owner видит обычный `pending_review` без infrastructure details.
- `changes_requested`/reject атомарно переводит export state в `discarded` и пишет
  high-priority idempotent object delete. Export не переиспользуется новой revision;
  late worker completion не меняет discarded state и удаляет созданный object через
  retry/DLQ/alert.
- Submit использует автоматическое standing consent первого `/start`, показывает
  CC BY 4.0 reminder/link и не имеет отдельного agreement gate.
- Sanitized GPX опубликованного track доступен анонимно через canonical download URL:
  endpoint не создаёт account/session, повторно проверяет publication status,
  до lookup/signing применяет 30 requests/min на normalized IP с burst 10 и выдаёт
  5-minute presigned redirect. `404` также расходует лимит; session его не повышает,
  S3 GET не учитывается. Private/archived/moderator-removed state отвечает одинаковым
  `404`; bot не публикует signed URL.
- Sanitized object не шифруется приложением и остаётся в private bucket. Presigned
  HTTPS URL отдаёт exact object напрямую без backend decrypt proxy; transparent
  provider-side SSE/disk encryption допустимы как deployment control и не меняют
  bytes/API.
- Canonical download `302/404/429/5xx` всегда имеет `Cache-Control: no-store`; CDN
  cache и `ETag` для route отсутствуют. Каждый request повторяет limit/publication
  checks, а signed S3 URL живёт отдельно только пять минут.
- Canonical `/tracks/:track-id/download` всегда разрешает только current published
  revision. Старые revision exports не имеют public route и eligible для storage
  retention cleanup; approval новой revision переключает тот же URL без изменения
  внешней ссылки.
- Sanitized S3 object задаёт `Content-Type: application/gpx+xml; charset=utf-8` и
  `Content-Disposition: attachment` с ASCII slug/UUID filename, поэтому эти headers
  действуют после presigned redirect.
- Sanitized object и final response содержат exact uncompressed XML bytes без
  `Content-Encoding`, gzip/ZIP или второго representation; filename заканчивается
  на `.gpx`.
- Download filename детерминированно строится из final Name: NFKD, lowercase ASCII
  alnum, collapse остальных runs в `-`, trim/max 64, fallback `track`, затем первые
  8 lowercase hex track UUID и `.gpx`. Transliteration и raw upload filename не
  используются.
- Переход current export в superseded/non-public пишет high-priority idempotent S3
  delete с retry/DLQ/alert. После delete ранее выданный signed URL не работает; при
  задержке доступ живёт не дольше исходных пяти минут. Отдельного download proxy нет.
- Htmx-форма `/tracks/new`: upload создаёт async job; polling открывает private draft
  preview; пользователь подтверждает final summary и отправляет revision на
  moderation. Submit требует непустой final name и явно выбранный primary activity;
  CC BY 4.0 показывается reminder/link без отдельного действия, description и
  остальные metadata optional.
- Любой GPX attachment в private one-to-one chat после quota/three-slot check
  автоматически начинает upload flow: backend скачивает file_id через provider API
  и использует тот же S3+parse+DB pipeline. `/upload` только предлагает прислать файл
  и не создаёт обязательный pending mode; attachment другого типа не создаёт job.
  Обязательные metadata и отправка на moderation завершаются в chat; web deep-link
  optional. Group/channel upload не создаёт job или draft.
- До трёх chat upload flows могут идти параллельно; каждый status/control/prompt
  связан с конкретным `upload_job_id`, неявного current upload нет.
- После parse metadata редактируются из одной draft summary card без обязательного
  порядка: Name/Activity отмечены required, actions открывают bound prompts для любых
  полей. Submit с пропусками ничего не меняет и перечисляет их; готовый draft
  переходит к final summary и CC BY 4.0 confirmation.
- Initial Name default выбирается из file-level `<metadata><name>`, затем из
  единственного distinct непустого `<trk><name>`, затем из безопасно
  нормализованного attachment filename stem. Несколько разных track names дают
  warning и не выбираются/не склеиваются. Любой default остаётся неподтверждённым
  draft field; raw filename отдельно не хранится и не публикуется. Без источников
  Name пустой.
- Initial Description default выбирается из file-level `<metadata><desc>`, затем из
  единственного distinct непустого `<trk><desc>`. Несколько разных track descriptions
  дают warning, не выбираются/не склеиваются и оставляют optional Description пустым.
  Выбранный default записывается только в owner UI locale на момент upload.
- Attachment caption не становится Description автоматически и не копируется в
  domain metadata. Description default берётся только по принятому GPX precedence или
  из текста, явно введённого через bound prompt; caption существует лишь в обычном
  raw-webhook retention.
- GPX `<desc>` без locale записывается в `description[owner_ui_locale_at_upload]` как
  неподтверждённый default. Autodetect/translation/duplication между `ru` и `en` нет;
  последующая смена UI locale не переносит существующий text.
- Presentation выбирает Description в UI locale, затем другую доступную ветвь с
  видимой language label; без обеих ветвей секция скрыта. JSON API возвращает полный
  `description` map, не переводя и не подменяя branches.
- Sanitized GPX сериализует обе непустые description branches в один deterministic
  plain-text `<desc>` блоками `[ru]`, затем `[en]`; пустой map не создаёт element.
  Viewer-specific exports, translation и custom XML extensions отсутствуют.
- Sanitized GPX использует только стандартные GPX 1.1 поля provenance/license:
  root `creator="TrailBase revision:<uuid>"`, author display name в
  `<metadata><author><name>`, CC BY 4.0 в
  `<metadata><copyright author="..."><license>`, canonical track URL в
  `<metadata><link href="...">`, а пользовательские Name/Description в
  `<trk><name>`/`<trk><desc>`. Provider identity, internal user ID, raw filename и
  application extensions отсутствуют.
- Sanitized export всегда содержит один `<trk>` на revision и один `<trkseg>` на
  каждый canonical `MultiLineString` component в deterministic order. Исходная
  multi-`<trk>` grouping не восстанавливается; route-only input также становится
  `<trk>`, а не `<rte>`.
- Sanitized `<trkpt>` содержит `<ele>` только при валидном исходном elevation этой
  сохранившейся после trimming point. Missing elevation остаётся missing; zero,
  interpolation и smoothed/LTTB values не экспортируются независимо от derived
  profile coverage.
- Export numeric format использует deterministic `HALF_EVEN`: lat/lon максимум
  7 decimal places, elevation максимум 2, decimal dot, plain notation без exponent
  и trailing zeros; negative zero нормализуется в `0`.
- Весь export одной revision byte-identical: UTF-8 без BOM, fixed XML declaration,
  GPX 1.1 namespaces и element/attribute order, compact XML без insignificant
  whitespace, один final LF и никаких runtime timestamps/random values.
- Export не добавляет durable SHA-256/byte-size columns. S3 PUT использует signed
  actual payload SHA-256 и только после checksum-validated success переводит state в
  ready; deterministic bytes покрываются golden tests и не становятся HTTP `ETag`.
- `<metadata><bounds>` не экспортируется: viewer получает bounds из track points, без
  дублирования geometry и неоднозначного antimeridian min/max.
- Activity, difficulty и tags не записываются в `<trk><type>` или
  `<metadata><keywords>`; их versioned taxonomy доступна по canonical URL/JSON API,
  а GPX mapping vocabulary в MVP отсутствует.
- Один attachment с одним или несколькими `<trk>` создаёт один draft, без
  automatic split. Каждый валидный `<trkseg>` становится отдельным компонентом
  canonical `MultiLineString` в document order. Segment manifest не создаётся;
  исходная `<trk>/<trkseg>` hierarchy остаётся только в original raw GPX.
- Parse result сохраняет только `source_track_count`, `source_route_count` и
  `valid_segment_count`. Несколько source elements выбранного track/route geometry
  kind дают неблокирующий warning с counts в draft/final summary рядом с map preview;
  отдельного confirmation до metadata editing нет, а final submit подтверждает весь
  snapshot.
- Route-only GPX следует тем же правилам: один attachment создаёт один draft, каждый
  `<rte>` становится component. Name выбирается из metadata, затем единственного
  distinct route name, затем filename stem; Description — из metadata, затем
  единственного distinct route description, иначе пусто. Разные values дают warning
  без выбора или concatenation.
- Для каждого принятого attachment bot сразу создаёт одну status card и best-effort
  редактирует её на стадиях download/validation/parse. Terminal success превращает
  card в draft summary/actions, failure — в безопасную причину и retry action. При
  provider edit failure отправляется одна replacement card без повторного domain job.
- После исчерпания automatic retries transient failure позволяет atomically
  reacquire-ить slot и повторно поставить тот же `upload_job_id` в очередь,
  переиспользуя сохранённый raw object. Permanent validation/limit/geometry error
  просит новый исправленный attachment; отсутствующий/очищенный raw также требует
  повторной отправки файла.
- Parsed draft, ожидающий required metadata или moderation submit, продолжает занимать
  один из трёх slots; slot освобождается после submit, cancel или terminal
  intake/parse failure.
- Cancel после parse закрывает chat flow, но сохраняет private draft/raw под quota;
  продолжение доступно через `/drafts` или web, delete — отдельное подтверждённое
  действие.
- `/drafts` resume атомарно занимает свободный slot; при полном лимите state не
  меняется и bot показывает три active flows.
- На draft действует один global lease для Telegram/Max/web. Явный takeover сохраняет
  slot, меняет lease generation и инвалидирует старые prompts/controls.
- Takeover требует authenticated owner и explicit confirmation; web проверяет CSRF,
  bot — private-chat identity, fresh bot auth не требуется.
- После takeover старый bot prompt best-effort теряет controls; edit failure не
  откатывает lease. Stale web mutation получает `409` и reload/resume action без
  realtime transport.
- Post-parse lease закрывается после 24 часов без успешной user mutation, освобождая
  slot без удаления draft/raw; reads/polling/worker events deadline не продлевают.
- PostgreSQL хранит durable lease generation/interface/activity и slot accounting;
  acquire/takeover/metadata/slot transitions транзакционны. Valkey содержит только
  ephemeral prompt/control tokens.
- Миграция upload flows добавляет `slot_no` 1..3, `closed_at` и partial unique indexes
  на active `(user_id, slot_no)` и active non-null `draft_id`.
- Upload/resume transaction перед acquire закрывает expired post-parse leases,
  увеличивает generation и только затем выделяет slot; janitor использует ту же
  operation как fallback.
- Slot-changing transactions сериализуются через `SELECT ... FOR UPDATE` на `users`
  row и lock order `user -> flow -> draft`; advisory locks не используются.
- Pre-parse cancel ставит `cancel_requested_at`, меняет generation и удерживает slot
  до terminal `cancelled`; worker проверяет cancel перед S3/parse/revision commit.
- Parse worker при deactivated owner может завершить started job только private draft;
  он не submit-ит revision и не создаёт publication side effects.
**Acceptance**:
- Web и bot принимают несжатый GPX 1.0/1.1 до 10 MiB и максимум 250 000 points.
- Успешный parse создаёт private revision snapshot с 2D MultiLineString, metrics,
  user-confirmed activity и S3 references.
- Raw object нельзя получить через public/owner API или presigned URL; его
  читают только backend workers. Единственный user-facing GPX download —
  опубликованный sanitized export.
- S3 raw object byte-identical исходному принятому attachment: application encryption,
  compression, normalization и иное преобразование bytes отсутствуют.
- Parsed raw не истекает отдельно от draft/track, остаётся учтённым в raw quota и
  удаляется при физическом delete/purge owning entity. Janitor не удаляет referenced
  raw по возрасту; его 24-hour rule относится только к incomplete/orphan objects.
- Две revisions одного track, созданные из одного existing raw без нового upload,
  ссылаются на один immutable object. Quota считает его один раз, удаление одной
  revision не удаляет object при оставшейся durable reference; новый upload всегда
  создаёт отдельный object даже для byte-identical content.
- Storage metadata raw находится только в `raw_objects`; upload job и
  каждая derived revision ссылаются на одну row через `raw_object_id`. Quota не
  умножается на число revision references, cleanup не удаляет referenced object.
- Physical raw delete блокируется retained revision FK. Terminal job history не
  считается pin и после raw delete получает `raw_object_id = NULL`; retryable
  transient job удерживает raw только до incomplete deadline.
- Concurrent reparse-reference insert и cleanup имеют один winner: обе операции
  блокируют owner/raw rows, а reference insert разрешён только для `ready`.
  `delete_pending` нельзя вернуть в `ready`; FK `ON DELETE RESTRICT` ловит ошибочный
  physical delete referenced row.
- Raw row проходит только `pending -> ready -> delete_pending -> physical row
  delete`; abandoned `pending` может перейти прямо в `delete_pending`. Revision не
  может ссылаться на non-`ready` row. Отдельных raw `error`/`deleted` states нет:
  upload failure виден в job, cleanup failure — в outbox retry/DLQ.
- Cleanup command не содержит object key/HMAC и восстанавливает их только
  по `raw_object_id`. Replayed command после уже удалённой row или S3 `404` успешно
  завершается; unexpected state/FK не вызывает delete.
- Concurrent upload quota нельзя обойти: до PUT каждый `pending` занимает 10 MiB,
  после validation contribution уменьшается до actual byte size, а
  `delete_pending` немедленно освобождает logical quota независимо от S3 cleanup lag.
- Два concurrent uploads одного owner сериализуются owner row lock: оба не могут
  пройти проверку одной старой суммы. Quota вычисляется из indexed `raw_objects`
  rows, не из denormalized user counter.
- GPX с несколькими `<trk>` создаёт один draft. Границы `<trkseg>` сохраняются как
  отдельные components в deterministic document order; отдельная manifest
  table/JSONB и вторая копия coordinates не создаются.
- Multi-track draft/final summary показывает source-track и valid-segment counts и
  map preview, но не блокирует редактирование отдельным confirmation. Cancel позволяет
  вместо него загрузить каждый catalog track отдельным файлом.
- Route-only GPX с несколькими `<rte>` также создаёт один draft и показывает
  неблокирующий source-route/valid-segment warning. Metadata defaults используют тот
  же distinct-value precedence, подставляя `<rte><name>/<desc>` вместо track fields.
- Sanitized GPX для track и route input имеет один `<trk>` с final submitted
  Name/Description и количеством `<trkseg>`, равным числу canonical geometry
  components; порядок segments совпадает с component order.
- Каждая exported point сохраняет только собственное валидное source elevation;
  missing `<ele>` не синтезируется, а 90% profile threshold не удаляет доступные
  per-point values и не добавляет отсутствующие.
- Повторная генерация той же revision создаёт byte-identical point numeric text:
  lat/lon не больше 7 decimals, elevation не больше 2, без exponent/trailing zeros
  или negative zero.
- Повторная generation одной revision создаёт byte-identical GPX object целиком;
  serializer не зависит от locale, platform newline, map iteration order, clock или
  randomness.
- PostgreSQL не хранит export hash/size. Checksum mismatch или PUT failure оставляет
  export не-ready и проходит обычный retry/DLQ; canonical route остаётся
  `no-store` без `ETag`.
- Final S3 GET отвечает GPX MIME type и attachment disposition с принятым ASCII
  filename; browser не пытается inline-render XML после redirect.
- Downloaded bytes совпадают с deterministic stored XML object; transparent/explicit
  compression и archive extraction в MVP отсутствуют.
- Sanitized export хранится как application-plaintext exact XML в private bucket и
  скачивается по presigned HTTPS URL без decrypt proxy. Provider-side SSE/disk
  encryption, если включено при deployment, прозрачно для bytes/API.
- Non-Latin-only Name получает `track-<8-hex-track-uuid>.gpx`; ASCII slug ограничен
  64 символами и не содержит header separators/control chars.
- Sanitized GPX не содержит `<metadata><bounds>`; корректность export не зависит от
  отдельного derived bbox или antimeridian convention.
- Sanitized GPX не содержит classification codes в `<trk><type>`/keywords; activity,
  difficulty и tags остаются доступны на track page и в JSON API.
- При наличии timestamps canonical duration по умолчанию использует moving с fallback
  на elapsed; manual/unknown override сохраняет исходные moving/elapsed metrics.
- Manual duration вне `1..31_536_000` отклоняется; более чем 10× расхождение с
  default auto candidate сохраняется только после explicit warning confirmation;
  ровно 10× и отсутствие auto candidate warning не вызывают.
- Подтверждённый duration outlier остаётся подтверждённым после resume/takeover, но
  stale control, изменённое manual value или reparse не могут переиспользовать
  acknowledgement для других comparison inputs.
- Moderator видит acknowledged duration outlier и comparison inputs; наличие flag
  само по себе не меняет moderation outcome или порядок очереди.
- Moderator не может исправить duration pending revision; после
  `changes_requested` авторский resubmit создаёт новый immutable snapshot, а прежний
  остаётся неизменным.
- Новый revision submit использует одноразовый standing CC BY 4.0 consent и не требует
  повторного acceptance.
- Anonymous visitor без account/session может скачать только public sanitized GPX
  через canonical URL; signed redirect живёт пять минут, а непубличные states не
  различаются внешним `404`.
- Download limiter допускает 30 attempts/min на normalized client IP с burst 10,
  считает `404` до publication lookup, возвращает `429 Retry-After` и не повышается
  browser session; redirected S3 GET не входит в budget.
- Ни один canonical download response не кэшируется: `no-store`, без `ETag`/CDN.
  Повторный request после revision switch/removal не использует старый redirect.
- Revision-specific public download отсутствует: после смены `current_revision_id`
  canonical URL выдаёт новый export, а superseded/privacy-pre-trim/removed snapshot
  нельзя получить старым TrailBase URL.
- После supersede/removal старый sanitized object удаляется high-priority cleanup;
  failure виден в DLQ/alert, а остаточный signed доступ ограничен five-minute TTL.
- Approve не может создать published revision без готового private sanitized export.
  Export error оставляет item pending и прежний current public; успешный approve
  атомарно переключает DB pointers без S3 generation внутри transaction.
- До approval owner/moderator не могут скачать private sanitized export и не
  получают presigned URL; object доступен только backend workers. После approval
  скачивание идёт исключительно через canonical current-revision route.
- Pending/error export item виден moderator-у и допускает
  `changes_requested`/reject, но не Approve. Manual retry не создаёт второй export
  job; owner-facing status не раскрывает S3/queue failure.
- После `changes_requested`/reject export становится discarded, private object
  удаляется, а новый immutable resubmit создаёт собственный export. Late completion
  не может вернуть discarded export в ready.
- `geometry_simplified_z11/13/15` populated для revision с tolerances 40/10/2 м.
- GPX attachment в private chat без предварительного `/upload` после quota/slot
  checks создаёт upload flow и draft-track с распарсенными auto-derived. `/upload`
  только показывает инструкцию, а attachment другого типа не создаёт job. User
  подтверждает final name и primary activity и отправляет revision на moderation без
  web session. Summary показывает CC BY 4.0 reminder/link; description optional,
  auto-derived metrics не требуют отдельных подтверждений.
- Один bot upload использует одну status card на `upload_job_id`: переходы стадий
  редактируют её best-effort, success показывает draft actions, failure — безопасную
  причину/retry. Ошибка provider edit может создать replacement card, но не второй
  upload job.
- Explicit retry после исчерпания transient attempts повторно занимает slot и
  requeue-ит тот же job не более одного concurrent attempt; сохранённый raw не
  загружается повторно. Permanent file error и отсутствующий raw требуют новый
  attachment.
- Metadata text применяется только как reply на prompt конкретного upload job; иначе
  bot просит выбрать один из active uploads.
- Name, Activity и optional metadata можно менять в любом порядке из draft summary
  card. Submit без Name/Activity не создаёт moderation revision и возвращает точный
  список пропусков; заполненный draft показывает final summary/license confirmation.
- Name precedence детерминирован: file-level metadata name, затем единственное
  distinct непустое track name, затем нормализованный filename stem. Разные track
  names не выбираются и не concatenated; warning виден пользователю. Default требует
  final confirmation, raw filename отсутствует в storage metadata/public export.
- Description precedence детерминирован: file-level metadata description, затем
  единственное distinct непустое track description. Несколько разных values дают
  warning и пустой optional field без concatenation; выбранный default попадает
  только в upload-time owner locale.
- Caption входного attachment не появляется в draft/public Description. Только GPX
  `<desc>` и explicit bound Description input могут заполнить это поле.
- GPX `<desc>` без language marker попадает только в сохранённый UI locale owner на
  момент upload; смена UI locale не меняет эту ветвь, а другой перевод добавляется
  explicit metadata action.
- Web/bot detail показывает Description fallback на вторую locale с явной меткой
  языка; JSON response сохраняет обе branches, а пустой map не создаёт секцию.
- GPX export с двумя descriptions содержит один XML-escaped `<desc>` с deterministic
  `[ru]`/`[en]` blocks; один approved revision не порождает варианты по viewer locale.
- GPX export использует root `creator="TrailBase revision:<uuid>"` и стандартные
  metadata author/copyright-license/link для immutable author display name, CC BY 4.0
  и canonical track URL; provider/internal identity, raw filename и application
  extensions отсутствуют.
- Четвёртый upload не принимается, пока один из трёх flows не освободит slot.
- Cancelled-after-parse draft остаётся доступным owner и не удаляется автоматически.
- Resume не позволяет превысить три active upload flows.
- Metadata mutation со stale lease generation отклоняется и не перезаписывает более
  свежие изменения из другого interface.
- Подтверждённый takeover из web/chat работает без дополнительного bot round-trip.
- Старый interface не может изменить metadata после takeover; provider notification
  failure не влияет на committed generation.
- Idle-expired draft можно позднее снова открыть через обычный slot-checked resume.
- Потеря Valkey инвалидирует prompts/controls, но не меняет lease ownership или число
  занятых slots.
- Деактивация во время parse не оставляет orphan data: job завершается private draft
  либо обычным terminal failure/cleanup, но не переходит в moderation автоматически.
- Конкурентные upload/resume не могут создать четвёртый active flow или два active
  leases одного draft даже без предварительного `COUNT`.
- Expired lease не вызывает ложный отказ по полному лимиту, даже если janitor ещё не
  запускался.
- Конкурентные lifecycle operations одного user сериализованы, но не блокируют flows
  других users.
- Cancel-wins не создаёт draft и чистит temporary raw; commit-wins применяет
  недеструктивный post-parse cancel.
- Reparse не переписывает опубликованный snapshot; он создаёт новую revision с
  versioned algorithms.

**Non-goal**: классификация/difficulty/season (M07), POI autodetect (M05), фото (M08).

---

## M04 — Catalog Render (Map Island)

**Объём**: ключевой срез. MapLibre-остров + OSM-raster basemap + adaptive GeoJSON delivery + bridge glue.

**Файлы/компоненты**:
- `src/trailbase/render/map.clj` — Hiccup partial для контейнера MapLibre (с `hx-disable` / помечен вне swap-области).
- `resources/public/js/map.js` — MapLibre init, raster source (OSM tiles), POI-cluster source + track-polyline source, event handlers (`moveend`, `click`).
- `resources/public/js/bridge.js` — alpine store `Alpine.store('map')`, htmx.ajax из map events, setData на track source.
- Сервер: `/api/tracks.geojson?bbox=&zoom=` → honeysql query: `ST_AsGeoJSON(ST_Force2D(geometry_simplified_zNN))` по zoom-aware колонке + `(WHERE ST_Intersects(geometry, bbox))` GIST.
- Сервер: `/api/locations/clusters.geojson?bbox=&zoom=` → server-side cluster для low-zoom (см. M05; в M04 — заглушка, возвращает по одному центроиду на track как POI-замену).
- Sidebar (htmx-зона): `/tracks` → partial; альпin-bridge: hover трек в списке → `map.setFeatureState({highlight: true})`.
- Track detail view: `/tracks/:id` → partial с мини-картой (на full zoom) + метаданные.

**Acceptance**:
- Карта показывает OSM-raster basemap.
- z0–12 получает server-side POI clusters; z13/z14/z15 используют соответственно
  z11/z13/z15 simplified geometry.
- На z16+ z15 остаётся context layer, а full geometry загружается только для
  выбранного track.
- Hover трека в sidebar → полилиния на карте стилизована (highlight).
- Performance: 5000 треков в тестовой выборке — pan/zoom без lag (>30 FPS) в Chrome.

**Non-goal**: POI синтаксис (M05), real search (M06), классификация (M07).

---

## M05 — POI Gazetteer

**Объём**: versioned location catalog, semantic categories, revision annotations,
autodetect, moderated OSM import и deterministic cluster endpoint.

**Файлы/компоненты**:
- Миграции создают `locations`, `location_revisions`, category dictionary,
  revision-location links и multiple ordered occurrences.
- `src/trailbase/locations.clj` — CRUD для модератора, list/get, autodetect-along-track (honeysql with `ST_DWithin` / `ST_Intersects` radius-aware), cluster endpoint.
- ОSM-import pipeline: `src/trailbase/import/osm.clj` — Overpass API fetch by osm_id/type, кэш name/geometry в `locations`, provenance record.
- Autodetect создаёт revision-location annotations с confidence/status. High-confidence
  links к approved locations одобряются автоматически; остальные идут в moderation.
  Новая POI всегда создаётся через заявку.
- Low-zoom cluster endpoint использует global hex-grid в EPSG:3857; MapLibre
  client-side clustering отключён.
- Bot уведомляет о завершении autodetect и ведёт deep-link в web preview; POI CRUD в
  боте не дублируется.
- Мoderator UI: `/moderation/locations` — список заявок, approve → creates location → re-runs autodetect binding.

**Acceptance**:
- После загрузки трека backend autodetect-ит POI вдоль трека (radius-aware), показывает их в загрузчик-форме.
- Загрузчик может предложить новое POI через форму (имя, тип, координаты или клик на карте) — заявка в модерацию.
- Модератор одобряет заявку → location revision публикуется → annotations
  пересчитываются.
- `/api/locations/clusters.geojson?zoom=10` возвращает ≤500 кластер-точек для bbox.
- Поиск «треки через локацию X» работает по approved revision-location annotations.
- OSM-import: модератор вводит osm_id+type → backend fetches Overpass → создаёт location с cached name/geometry.

**Non-goal**: модерация тегов (M07), фото POI.

---

## M06 — Search

**Объём**: единый PostgreSQL search engine — текст × гео × фасеты × POI-join. Instant-search.

**Файлы/компоненты**:
- Миграции `006-search.up.sql`:
  - GENERATE ALWAYS columns: `ts_description_ru to_tsvector('russian', description->>'ru')`, `ts_description_en to_tsvector('english', description->>'en')`, GIN индексы.
  - `pg_trgm` extension для name fuzzy matching, триграм-индекс на `name`.
  - compound indexes на `(activity_type)`, `(difficulty)`, `(season)`, `(duration_s)` buckets.
- `src/trailbase/search.clj` — honeysql composable query builder: combine text
  (`tsvector` + `pg_trgm` fallback), geo bbox, facets и approved POI annotations.
- `/search` отдаёт HTML partial, `/api/v1/search` — JSON; оба используют opaque
  HMAC-protected keyset cursor и server-side disjunctive facet counts.
- Telegram/Max `/search` использует тот же domain search service, filters и keyset
  pagination, адаптированные к chat controls; browser session не требуется.
- Group/channel `/search` использует public principal: только published public tracks,
  без account creation, персональных данных и сохранения history/settings.
- Private-chat `/search` linked deactivated identity использует тот же stateless
  public principal без private results, account-specific facets или history/settings.
- Group-search controls связаны с provider/chat/message/requester; только инициатор
  запроса может менять filters или page общего результата.
- Channel search без стабильной requester identity возвращает статическую первую
  страницу без controls и предлагает продолжить в private chat.
- Search controls в private/group chats истекают через 15 минут от создания без
  продления; expired callback не редактирует старое сообщение.
- Chat-search callback содержит только случайный 128-bit opaque ID; binding,
  query/filters/cursor и absolute expiry хранятся в Valkey.
- Каждый успешный control callback атомарно заменяет текущий opaque ID новым с тем же
  исходным `expires_at`; только победитель ротации редактирует message.
- Transient/ambiguous provider edit повторяется с тем же новым ID; terminal failure
  удаляет новый state без rollback старого.
- Search-result edit имеет максимум пять total attempts с backoff/jitter; retry только
  для network/timeout, `429` и `5xx`, не позже исходного `expires_at`.
- Callback acknowledgement отправляется после binding/expiry validation и atomic
  rotation, до query/edit; `bot-worker` обновляет result асинхронно.
- Query timeout/transient failure после ACK удаляет новый state без rollback,
  сохраняет прежний result content и terminal edit убирает controls.
- Instant-search: `hx-get` on `keyup changed delay:300ms` → htmx partial swap `#results` и `#facets`.
- Location-constraint: `location_id=N` → join approved revision annotations.
- Geography-aware: bbox в segmented radio — server postgis geometry comparison.

**Acceptance**:
- Free-text поиск по track name/description: tsvector + триграмы, опечатки 1-2 символа учитываются.
- Фильтр `activity=hike`, `difficulty=T3`, `season=winter`, `duration_min=3600 & duration_max=21600` (1-6h) — composable AND.
- Комбинированный: текст + фасеты + bbox одновременно — один SQL-запрос через CTE.
- POI-filter: `location_id=5` → список треков через локацию 5.
- Instant-search: ввод 3+ символов → partial swap без перезагрузки страницы; сервер откликивается <200ms на индексированных данных.
- `#facets` обновляется server-side aggregation (activity counts, difficulty distribution) при каждом изменении фильтра.
- Мультиязычность: `description` jsonb с языковыми ветвями; tsvector собирается из всех языков, ranking по combined.
- Bot `/search` возвращает те же результаты и permission-filtered facets, что web/API,
  с постраничной навигацией в chat.
- Group/channel search никогда не возвращает private/unlisted tracks; bot upload и
  state-changing commands остаются private-chat-only.
- Search от deactivated identity возвращает только public results и не создаёт новый
  account или persistent personal history/settings.
- Нажатие group-search control другим участником не меняет результат и предлагает ему
  запустить отдельный `/search`.
- Channel search без requester identity не создаёт callback state; ссылка для
  продолжения в private chat не содержит auth token.
- Через 15 минут search control предлагает повторить `/search` и не меняет старый
  result message.
- Потеря Valkey callback state безопасно инвалидирует controls; query и requester
  identity отсутствуют в provider callback payload.
- Повторный или конкурентный callback со старым ID считается stale и не может
  перезаписать новый search result.
- После исчерпания provider-edit retries result становится неинтерактивным и предлагает
  повторить `/search`.
- Permanent `4xx` не retry-ится; исчерпанный ephemeral edit не попадает в DLQ/replay.
- Stale/foreign/expired callback получает нейтральный acknowledgement, не ротирует
  state и не изменяет общее сообщение.
- При query failure старый result остаётся видимым без controls и предлагает повторить
  `/search`; terminal edit использует общий provider retry policy.

**Non-goal**: external search engine (Meilisearch), semantic search (pgvector).

---

## M07 — Classification & Moderation

**Объём**: tag dictionary + очереди заявок тегов; difficulty/season/duration facet wiring; модераторский API.

**Файлы/компоненты**:
- Миграции `007-tags.up.sql`:
  - `tags(id, key, label_i18n jsonb, parent_id, status enum approved/pending/rejected, created_by, approved_by)`.
  - `revision_tags(track_revision_id, tag_id, source enum user/moderator/derived)`.
  - `tag_requests(id, requested_tag_label, requested_by, status, declined_reason, created_at, reviewed_by, reviewed_at)`.
- `src/trailbase/tags.clj` — словарь (list, CRUD модератор), запрос нового тега, одобрение, привязка к треку.
- `src/trailbase/walks/facets.clj` — обёртка для difficulty (per-activity lookup: SAC для hike, MTB-scale для bike, 3-level fallback), season bitmask, duration handling.
- UI: `/tracks/:id/edit` — частичная форма с тегами (autocomplete из approved-словаря) + фасетами (difficulty по current activity_type, season checkboxes, duration manual override if `duration_source=manual`).
- Moderation queue: `/moderation/tags` — список pending tag_requests; approve → создаёт `tags` row; assign to track (если request был из track edit).
- M03 auto-suggestion теперь стersistие tags suggestions помимо activity_type (basic heuristic — `alpine` if high elev/hike, `loop` if start≈end point).

**Acceptance**:
-Approved-теги видны как autocomplete при редактировании трека.
- Юзер без модераторской роли не может добавить новый тег из ниоткуда; может только предложить → queue.
- Модератор видит очередь, approve/reject с reason; approve создаёт tag в словаре.
- Difficulty widget меняется по activity_type (hike → SAC-T1..T6 selector; bike → S0..S5; fallback → 3-level).
- Season bitmask хранится и фильтруется по multiple bits (`season & 1`, `season & 2`, ... — winter=8 etc.).
- M06 search facet-агрегация теперь использует настоящие tags/difficulty/season — end-to-end работает.

**Non-goal**: ML-based tag suggestion (basic heuristic only).

---

## M08 — Photo + Elevation

**Объём**: фото трека (S3, presigned), профиль высоты (alpine island), gallery lightbox.

**Файлы/компоненты**:
- Миграции `008-photos.up.sql` — `track_photos(id, track_id, s3_uri, caption, taken_at?, seq)`; optionally geometry from EXIF for map EXIF-plots.
- `src/trailbase/photos.clj` — upload flow (multipart → S3 via aws-simple-sign+hato → insert row), presigned URLs (rotating expiry), delete.
- Htmx / alpine integration: `/tracks/:id/photos` partial-ul — список + upload-form (`hx-post` with progress indicator).
- Alpine island for gallery lightbox: `x-data="{open, index}"` + keyboard nav; lazy-swipe for many photos.
- Elevation profile chart использует LTTB-profile до 2 000 samples из track revision;
  полный point array в PostgreSQL не хранится.
- Map EXIF-plots: photos with `geometry` from EXIF shown as markers on MapLibre (separate source).
- Presigned URL rotation: photo URLs expire; client refetches через htmx when needed.

**Acceptance**:
- Upload фото к треку → S3 → row в таблице; превью в галерее.
- Lightbox: click → fullscreen; ← → keyboard nav; Esc-close.
- Elevation profile: drawn from parsed GPX altitudes (if `<ele>` present); hover on chart → marker on map at corresponding track point; brush selection → zoom on chart for section.
- EXIF photos appear as markers on map (if EXIF contains GPS).
- Presigned URLs expire but no visible broken images — URLs rotate via alpine/htmx refetch.

**Non-goal**: video, live-tracking, social features.

---

## Cross-cutting concerns (投融资 не по срезам)

- **telemere** логирование: с M01; structured logs в каждом handler.
- **Malli schemas** для API валидации: с M01 (health schema) и расширяется каждый срез.
- **reitit coercion** полной цепочки middleware: с M01.
- **bot module** (raw Bot API over hato): Telegram и Max входят в M02; конец M02 —
  работающие identity и разрешённые chat operations через оба provider.
- **tests**: по одному integration test на каждый срез (smoke-level: upload fixture GPX, render map page, search returns expected track). Не TDD по слоям, smoke на end-to-end.
- **I18n**: user-selectable UI locales — `ru`, `en`; `simple` используется только как
  technical fallback для tags/POI names и full-text processing.

## Orders & Dependencies

- M01 — prerequisite всех.
- M02 — после M01 (moderate подмамины — bot, auth).
- M03 — после M02 (ownership требует messenger identity/account).
- M04 — после M03 (карта рендерит реальные tracks).
- M05, M06, M07, M08 — после M04; ** могут разрабатываться параллельно** (independent extensions; не блокируют друг друга кроме небольших hooks):
  - M06 search facet-aggregation нормально работает до M07 (tags NULL = no facets), а после M07 facets оживает.
  - M05 POI может отставать — M04 тогда использует fallback as-centroids; при готовом M05 cluster endpoint просто переключается.
  - M08 фото не зависит от M06/M07.

## Definition of Done (per slice)

- Acceptance criteria выполнены и продемонстрированы.
- Smoke-test green: запуск `bb run-dev` + fixture-data (через миграции/seed) → end-to-end scenario runnable.
- Миграции — мигration + rollback оба проходят на empty-DB.
- telemere logs хотя бы в ключевых handlers; Malli schemas на endpoint inputs.
- ADR-ссылки в PR-описании; новые ADR если design отклонился от roadmap.
