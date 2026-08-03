# Контракт: identities и account creation

Часть [Implementation Contract](../../IMPLEMENTATION-CONTRACT.md). При противоречии с другими документами действует этот контракт.

## 4. Bot-first authentication

### 4.1 Identity и создание аккаунта

- Telegram и Max — единственные identity и authentication providers. Email, телефон и
  отдельные recovery identities отсутствуют.
- Browser activation не требуется. Валидированная messenger identity достаточна для
  выполнения разрешённых chat operations и использования account без web session.
- Первый валидированный `/start` в private one-to-one chat от ранее неизвестной
  Telegram/Max identity атомарно создаёт active `users` row и первую
  `user_identities` row и минимальный `user_agreements` record. Повторная доставка и
  конкурентные `/start` идемпотентно resolve в тот же account через unique provider
  identity и не создают второе agreement; browser session для создания не нужна.
- Plain `/start` показывает, что такое TrailBase и бот, главное меню, обычную
  HTTPS-ссылку на актуальные правила и ссылку на CC BY 4.0, под которой размещаются
  пользовательские материалы. Сообщение объясняет: продолжая пользоваться ботом,
  пользователь автоматически соглашается с правилами, включая их будущие изменения.
  Отдельной кнопки «Согласен», pending agreement state и web-перехода нет;
  `/start <link-token>` остаётся отдельным linking flow.
- Главное меню Telegram/Max содержит пять основных действий в одном порядке:
  «Поиск», «Загрузить GPX», «Мои треки/черновики», «Настройки» и «Помощь».
  «Правила» и «Лицензия» доступны как вторичные ссылки и не входят в главное меню.
- «Настройки» содержит пять разделов: «Профиль» (`users.display_name` и язык),
  «Мессенджеры» (linked Telegram/Max identities и primary provider),
  «Уведомления» (category preferences), «Сессии» (active web sessions, revoke одной
  и logout-all) и «Аккаунт» (деактивация). Email, phone, password и recovery
  settings отсутствуют.
- Все пять разделов полностью работают в private Telegram/Max chat без web session.
  Profile/notification preferences изменяются прямо в chat; linking второго provider
  начинается там и завершается его `/start <link-token>`; web sessions можно
  просмотреть, revoke по одной или logout-all; account можно деактивировать. Уже
  принятые fresh-auth/confirmation requirements для sensitive operations сохраняются.
  Web settings UI является optional mirror тех же domain operations.
- «Мои треки/черновики» открывает в chat единый список всех принадлежащих
  пользователю tracks и upload/draft flows с явным пользовательским статусом:
  `draft`, `processing`, `pending_review`, `changes_requested` или `published`.
  Отдельного track lifecycle state `rejected` нет. Сначала идут элементы, требующие
  действия пользователя, затем остальные по `updated_at DESC`. Для просмотра списка
  web session не требуется.
- User-archived tracks исключены из обычного списка и доступны через вторичный фильтр
  «Архив» внутри «Мои треки/черновики». Карточка archived track показывает оставшийся
  срок до 30-дневной очистки и действие «Восстановить». Досрочный permanent delete
  archived track в MVP не предлагается.
- В MVP доступны только два list views: default «Все» без archived tracks и
  вторичный «Архив». Дополнительных filters по status нет; action-required entries
  выделяются уже принятым порядком и status label.
- User archive durable сохраняет непосредственно предшествующий lifecycle status и
  current revision. Restore до `purge_at` атомарно возвращает это состояние:
  опубликованный track снова становится `published` с той же approved revision без
  повторной moderation, непубликованный — прежним private status. Moderator removal
  этим flow не восстанавливается.
- «Восстановить» выполняется одним valid callback без дополнительного confirmation.
  Mutation атомарно проверяет owner, текущий `archived` status и `purge_at`.
  Повторная доставка операции идемпотентна и возвращает уже восстановленное состояние;
  foreign, stale или expired callback ничего не меняет.
- Действия карточки зависят от статуса: `draft` и `changes_requested` позволяют
  продолжить редактирование; `processing` — посмотреть прогресс или отменить;
  `pending_review` — только просмотреть; `published` — открыть, скачать sanitized
  GPX, начать новую revision или архивировать. Удаление draft и архивация track
  всегда требуют отдельного подтверждения.
- Список содержит 10 элементов на страницу и inline controls «Назад»/«Далее».
  Pagination использует deterministic server-side keyset по принятому порядку с
  stable kind/UUID tie-breaker. Provider callback содержит только случайный opaque
  ID; Valkey record хранит cursor и binding к provider/chat/message/requester.
  Нажатие с несовпадающим binding не раскрывает account data и не меняет сообщение.
- List controls имеют абсолютный TTL 15 минут от первого открытия без sliding
  продления. После expiry или потери Valkey state callback не редактирует старое
  сообщение: bot сообщает, что список устарел, и предлагает заново открыть «Мои
  треки/черновики».
- Кнопки/команды «Правила» и «Лицензия» в Telegram/Max возвращают соответствующие
  HTTPS-ссылки. Полные документы и их копии в chat не хранятся.
- Правила доступны по одной ссылке на актуальный текст и могут изменяться на этой
  странице в любое время. Изменения правил не версионируются в account model, не
  требуют повторного agreement, не меняют account/session capabilities и не создают
  специальные notifications или другие domain events.
- Автоматическое agreement при первом plain `/start` является standing consent на
  размещение пользовательских contributions под CC BY 4.0. Submit revision
  показывает license reminder/link, но не требует отдельного acceptance action.
- PostgreSQL `user_agreements` хранит минимальное append-only evidence:
  `user_id`, `accepted_at`, `source = /start`, SHA-256 показанного notice,
  `rules_url` и `license_url`. Unique constraint `user_id` обеспечивает не более
  одного agreement на account.
- Повторный plain `/start` не обновляет `accepted_at`, notice hash или URLs в
  `user_agreements`: record навсегда фиксирует первый `/start`. Bot снова показывает
  notice/menu и обновляет только обычный provider profile snapshot.
- Agreement records не содержат IP, raw session/link tokens или provider callback
  payload; PostgreSQL является единственным durable source of truth. Изменение
  содержимого по `rules_url` ничего в agreement record или account state не меняет.
- Account creation, выпуск web-session/link tokens и identity linking разрешены только
  в private one-to-one chat с ботом. Group/channel events не создают account, не
  изменяют identity state и не раскрывают account state; bot отвечает статической
  ссылкой или инструкцией продолжить в private chat без auth token.
- Дополнительная identity связывается с account только явно из уже
  аутентифицированного account context. Нельзя автоматически объединять accounts по
  имени, username или другим profile fields.
- Agreement принадлежит `user_id`, а не отдельной messenger identity. Успешно
  привязанная Telegram/Max identity сразу использует capabilities target account без
  нового notice record; evidence первоначального `/start` не меняется.
- Если identity уже принадлежит другому аккаунту, link отклоняется. Account merge
  не входит в M02.
- Active account всегда сохраняет хотя бы одну active Telegram/Max identity; отвязать
  последнюю identity нельзя.
- При unlink active identity физически удаляется. Audit хранит внутренний identity UUID
  и provider, но не `provider_user_id`.
- Основное публичное имя — редактируемое `users.display_name`. Оно инициализируется
  именем первой identity и затем не обновляется автоматически.
- Смена `users.display_name` применяется сразу без fresh auth и pre-moderation.
  Ввод нормализуется в Unicode NFC, внешние пробелы удаляются; пустое значение,
  control/newline characters и длина больше 200 Unicode code points отклоняются.
  Изменение audit-ится. Abuse обрабатывается общей moderation policy, а не блокирует
  каждую смену имени.
- Public track pages разрешают attribution через текущее `users.display_name`, поэтому
  после реактивации смена имени обновляет его на всех ранее published track pages.
  Отдельный snapshot author name в track/revision для web attribution не хранится.
- Provider display fields обновляются на `/start` и хранятся только как snapshot:
  `display_name`, `username`, `language_code`, ограниченный `profile_meta`.
- Provider avatar URL не сохраняется. Допустим только безопасный `provider_avatar_ref`;
  загрузка выполняется сервером.
- Публично отображаются только TrailBase display name и будущий TrailBase avatar.
  Provider usernames и identities не публикуются.
- Пользовательские UI locales в MVP — только `ru` и `en`. Язык при первом
  взаимодействии выбирается в порядке: сохранённая настройка, поддерживаемый
  `language_code` bot profile, поддерживаемый `Accept-Language`, затем `ru`. Ручной
  выбор применяется и к bot, и к web UI и больше не перезаписывается provider
  language. `simple` используется только как технический fallback для content
  labels/tags/POI и не является пользовательским locale.
