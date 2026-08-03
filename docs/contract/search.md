# Контракт: search

Часть [Implementation Contract](../IMPLEMENTATION-CONTRACT.md). При противоречии с другими документами действует этот контракт.

## 18. Search contract

- Search остаётся в PostgreSQL: tsvector/GIN, `pg_trgm`, PostGIS/GIST, btree facets и
  joins по approved revision annotations.
- Все public catalog principals и authenticated searches исключают tracks под active
  full track lock из results и facet counts.
- `/search` возвращает Hiccup/HTML для htmx; `/api/v1/search` — стабильный JSON. Оба
  используют одну domain search service.
- Web/bot search и detail выбирают Description по UI locale с fallback на вторую
  доступную ветвь и явной меткой языка. JSON API возвращает обе `ru`/`en` ветви без
  скрытого перевода или server-side подмены map.
- Telegram/Max `/search` использует ту же domain search service и выполняется без
  browser session. Bot адаптирует filters и pagination к chat controls; search semantics
  и permission checks не расходятся с web/API.
- В private chat search выполняется с permissions связанного account. В group/channel
  chat `/search` разрешён только как account-stateless read-only поиск по public
  published catalog: он не создаёт account, не возвращает private/unlisted tracks или
  персональные данные и не сохраняет user history/settings.
- Private-chat `/search` от linked deactivated identity использует тот же stateless
  public principal, а не permissions disabled account: только public published
  catalog, без history/settings, private results или account-specific facets.
- Filter/pagination controls у group-search результата принимает только requester,
  инициировавший конкретный `/search`. Callback cryptographically связывается с
  provider, chat, message, requester identity и search cursor/query state. Нажатие
  другим участником не меняет общий результат и предлагает запустить собственный
  `/search`.
- Если channel context не предоставляет стабильную requester identity, `/search`
  публикует только статическую первую страницу public results без filter/pagination
  controls. Для продолжения bot добавляет обычную ссылку в private chat без auth token
  и account state.
- Interactive search controls в private/group chats имеют абсолютный TTL 15 минут от
  создания результата без sliding refresh. После expiry callback не изменяет старое
  сообщение и предлагает повторить `/search`.
- Chat-search callback содержит только случайный 128-bit opaque ID. Valkey record хранит
  provider/chat/message/requester binding, query, filters, cursor и исходный absolute
  expiry; query и identity не попадают в provider callback payload. Потеря Valkey или
  записи инвалидирует controls без восстановления и предлагает повторить `/search`.
- Успешный filter/page callback атомарно проверяет binding/expiry, расходует текущий
  Valkey record и создаёт новый opaque ID с обновлённым search state, сохраняя исходный
  absolute `expires_at`. Только callback, выигравший ротацию, может редактировать
  result message; повторный или конкурентный callback со старым ID считается stale и
  message не меняет.
- Если rotation завершилась, а provider edit вернул transient error или ambiguous
  outcome, retry использует тот же новый ID и state. После исчерпания bounded retry
  новый Valkey record удаляется, старый не восстанавливается, а result message
  считается неинтерактивным и предлагает повторить `/search`.
- Search-result edit делает максимум пять total attempts с exponential backoff и
  jitter. Retry разрешён только для timeout/network errors, `429` и `5xx`; `Retry-After`
  соблюдается, только если следующая попытка укладывается в исходный `expires_at`.
  Остальные `4xx` terminal сразу. После terminal failure ephemeral edit не попадает в
  DLQ или поздний replay.
- Provider-specific callback acknowledgement отправляется сразу после проверки
  requester binding/expiry и успешной atomic rotation, до search query и result edit.
  Он подтверждает только приём нажатия; дальнейшую работу выполняет `bot-worker`.
  Webhook ingress `2xx` остаётся отдельным подтверждением доставки event.
- Stale, foreign или expired callback не ротирует state и не меняет result message;
  provider acknowledgement содержит короткое нейтральное объяснение.
- Если после ACK/rotation search query завершается timeout или transient error, новый
  callback state удаляется, а старый ID не восстанавливается. Bot сохраняет прежний
  result content, но terminal edit убирает controls и добавляет нейтральную инструкцию
  повторить `/search`; edit использует тот же provider retry policy.
- Instant search запускается с трёх символов. Явный submit длиной 1–2 символа выполняет
  только exact/prefix search без trigram.
- Exact/prefix tsvector results ранжируются выше fuzzy. `pg_trgm` — fallback, если
  точных результатов мало или score низкий.
- Все facet counts считаются server-side по полной выборке. Используется disjunctive
  faceting: каждый facet применяет все filters, кроме собственного.
- Pagination — HMAC-защищённый opaque keyset cursor, связанный с query hash.
  Default page size 20, maximum 100.
- При `q` default sort: relevance, completeness, publication time, UUID.
  Без `q`: newest, UUID. Дополнительные sorts: newest, length asc/desc,
  duration asc/desc. Popularity в MVP отсутствует.
- Поиск может быть привязан к map viewport. Режим «Искать в этой области» включён по
  умолчанию, отключаем пользователем и сохраняется в URL.
- Search SQL получает `statement_timeout = 800ms`, целевой p95 ниже 200 мс.
  Timeout возвращает `503` и retry affordance.
- Запрос медленнее 200 мс логирует fingerprint, facets, bbox size, row counts и
  duration без query text/user ID. Sampled `EXPLAIN (ANALYZE, BUFFERS)` выполняется
  только в test environment.

