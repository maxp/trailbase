# Контракт: notification delivery и provider health

Часть [Implementation Contract](../../IMPLEMENTATION-CONTRACT.md). При противоречии с другими документами действует этот контракт.

- Notification flood не подавляет records и не создаёт digest/coalescing: audit,
  web-inbox и outbox materialize-ятся немедленно для каждого claim. Dispatcher применяет
  provider-wide budget, per-target pacing и fair queues, учитывая provider
  `Retry-After`, чтобы noisy account не starve-ил другие targets. Local throttle
  оставляет delivery pending, не расходует attempt budget и сам по себе не ведёт в DLQ.
  Метрики low-cardinality: queue depth, oldest age и throttled count по provider, без
  target IDs; overflow не подавляет locked-on security facts.
- Provider `429 Retry-After` атомарно создаёт в PostgreSQL durable provider-specific
  `cooldown_until = max(current, parsed Retry-After)` для всех notification deliveries
  этого provider. Triggering Bot API call расходует одну attempt; waiting rows остаются
  pending без attempts, все loops/restarts соблюдают cooldown, другой provider работает.
  Missing/invalid header использует capped jittered fallback. Cooldown не блокирует
  ingress, `session_claimed` projection или web-inbox; метрика имеет только provider
  label.
- Provider adapters нормализуют delivery errors closed taxonomy:
  `provider_blocked|target_unreachable|rate_limited|transient|unclassified`. Только
  global auth/config `provider_blocked` открывает durable provider circuit: Bot API calls
  останавливаются, rows остаются pending без attempts, dispatcher readiness failed и
  alert; другой provider работает. `target_unreachable` terminal только для одной row,
  audit/web-inbox сохраняются. `unclassified` идёт в DLQ/alert без circuit. Raw provider
  body, target ID и credentials запрещены в logs, metrics и DLQ.
- `target_unreachable` не unlink-ит и не demote-ит identity, не меняет primary и не
  создаёт fallback/retarget для future notifications. Delivery error не доказывает
  потерю identity или намерение user изменить routing. Exact row завершается terminal
  без retry/retarget; audit/web-inbox сохраняются. Future events продолжают snapshot-ить
  explicit current primary даже при linked secondary. Только явный user unlink или
  change-primary меняет route; automatic duplicate/account mutation запрещены.
- `target_unreachable` ставит на identity durable generic
  `delivery_health=unreachable`; identity хранит только normalized health и observed
  timestamp без raw provider reason. Latest observed outcome под identity row lock
  побеждает concurrent stale result. Web «Мессенджеры» показывает owner-у generic
  warning и explicit change-primary/unlink actions. Successful ordinary send exact
  identity либо validated inbound user message от exact linked provider identity
  возвращает `reachable`; для inbound observation используется application server
  acceptance time, не provider timestamp. Unlink удаляет state. Health не влияет на
  auth, linking, primary selection/queued deliveries и не создаёт notification.
- `delivery_health=unreachable` не имеет TTL и manual dismiss: time passage, restart,
  закрытие provider circuit и смена primary не доказывают recovery. Для non-primary
  identity warning остаётся только в её card без account-wide banner. Recovery даёт
  только successful ordinary send exact identity или validated inbound user message
  от exact linked identity; physical unlink удаляет state, а relink создаёт новую
  identity с initial `unknown` без наследования старого failure.
- Inbound recovery принимает только validated user-authored private-chat message event
  от exact linked provider identity, включая command, text и media messages.
  `callback_query`, inline query, membership/service events и delivery receipts не
  являются новым сообщением пользователя и `delivery_health` не меняют.
- Provider event dedupe выполняется до inbound health mutation. Только первое accepted
  unique message ставит `reachable` и application acceptance timestamp; exact webhook
  replay получает idempotent acknowledgement без DB health update. Replay не может
  получить новый observed timestamp и победить более свежий `target_unreachable`.
- Accepted inbound message exact linked identity обновляет health и для deactivated
  account: channel reachability не является authorization. Mutation не reactivate-ит
  account, не создаёт session/notification и не обходит deactivation command policy;
  после будущей reactivation сохранённый health отражает наблюдавшийся канал.
- Inbound health recovery и unlink exact identity берут identity row lock. Unlink-first
  удаляет identity, поэтому inbound event не обновляет health, не relink-ит и не создаёт
  новую row. Inbound-first может поставить `reachable`, после чего unlink удаляет row
  вместе с health. Old provider identity не воскресает и не attach-ится к future relink.
- Concurrent inbound `reachable` и outbound `target_unreachable` сравниваются по
  application observation time, захваченному до ожидания identity lock: accepted-event
  time после dedupe для inbound и время получения normalized provider result для
  outbound. Под lock применяется только более новое observation; commit order,
  provider timestamp и send-start time не участвуют. При exact tie `unreachable`
  побеждает консервативно.
- Owner UI не показывает `delivery_health_observed_at`: Web/private-chat identity card
  имеет только actionable warning для `unreachable`, а `unknown|reachable` не получают
  diagnostic badge или timestamp. Поле остаётся internal ordering/operations data и не
  называется last activity/last successful delivery, которых оно не доказывает.
- Warning является derived UI projection current identity row, не standalone messenger
  delivery, domain notification, web-inbox или outbox. Web показывает его при owner
  view. Private-chat settings может показать warning только в response через другую
  working linked identity; message в ранее blocked exact bot сначала применяет inbound
  recovery, поэтому его собственная card уже не warning-ит. Если working identity нет,
  bot warning недоставим по определению и доступен только Web.
- Closed ru/en warning copy: RU — «Доставка в этот мессенджер недоступна. Разблокируйте
  бота, если нужно, и отправьте ему сообщение.»; EN — “Delivery to this messenger is
  unavailable. Unblock the bot if needed, then send it a message.” Copy одинаков для
  Web и card через другую identity, не раскрывает raw provider cause и расположен рядом
  с existing change-primary/unlink actions.
- Transition `unreachable -> reachable` не создаёт user-visible recovery confirmation:
  warning исчезает при следующем render/read identity card. Toast/flash, отдельный bot
  reply, domain notification, web-inbox, outbox и audit event отсутствуют. Обычный
  response на исходную command/message не меняется и не объявляет channel recovery.
- Уже открытая Web identity card не получает automatic inbound-recovery refresh:
  polling, SSE и WebSocket только ради health warning отсутствуют. Current page может
  оставаться stale до следующего обычного navigation/htmx card refresh; следующий
  server render читает authoritative identity row и убирает warning. Bot card
  обновляется при следующем user request. `Cache-Control: no-store` сохраняется.
- Единичный `target_unreachable` — обычный per-user state transition и operator alert не
  создаёт. Telemetry оставляет только aggregate metrics по provider и normalized outcome
  без identity/target labels. Alert разрешён только для provider-wide threshold/circuit
  condition, а не для блокировки бота одним пользователем.
- Provider-wide alert не вычисляется из количества rows с
  `delivery_health=unreachable`: persistent per-identity state может долго сохраняться
  для blocked user и non-primary identity и не отражает current provider health. Alert
  использует rolling provider-call outcome metrics и circuit state; periodic DB
  scan/gauge `unreachable` identities отсутствует.
- `target_unreachable` остаётся aggregate counter с labels `provider` и
  `normalized_outcome=target_unreachable`, но не входит в provider availability error
  rate, не вызывает circuit transition и не учитывается в provider-wide alert threshold:
  target-specific блокировка не является provider incident.
- `rate_limited` (`429`) — отдельный throttling/capacity signal и также не входит в
  provider availability error rate и не открывает circuit. Telemetry использует
  `normalized_outcome=rate_limited` series общего call-attempt counter, cooldown age и
  backlog metrics; alert допустим при sustained cooldown или росте backlog, отдельно от
  availability alert.
- Unit provider availability rate — каждый фактически выполненный Bot API call attempt,
  а не terminal outcome outbox row. Ответ или timeout каждого retry attempt учитывается
  отдельно; последующий success не стирает предыдущий transient failure. Delivery/outbox
  lifecycle имеет отдельные metrics. Calls, подавленные cooldown/open circuit и не
  отправленные provider-у, в denominator отсутствуют.
- `unclassified` не входит в provider availability error rate: это classification gap
  или adapter-contract drift, не доказанный provider failure. Aggregate counter с labels
  `provider` + `normalized_outcome=unclassified` и existing DLQ/operator alert
  сохраняются; circuit не открывается, rolling numerator учитывает только
  классифицированные availability failures.
- Triggering фактический call attempt с `provider_blocked` входит в availability
  numerator как реальная auth/config невозможность вызвать provider. Одновременно
  durable circuit, readiness failure и operator alert срабатывают немедленно, не ожидая
  rate threshold; подавленные после открытия circuit calls не входят в denominator.
- Rolling `transient` error-rate alert имеет minimum sample gate: он оценивается только
  после минимального числа фактически выполненных call attempts в rolling window, чтобы
  один timeout при низком traffic не создавал provider-wide incident. `provider_blocked`
  bypass-ит gate; suppressed calls sample не увеличивают.
- MVP использует rolling window 5 минут и minimum sample 20 фактических call attempts на
  provider. При sample `< 20` rate остаётся видимым в metrics, но alert не открывается;
  `provider_blocked` по-прежнему bypass-ит gate.
- `transient` alert открывается при error rate `>= 30%` в этом 5-minute window и sample
  `>= 20`. Numerator содержит `transient` attempts; denominator — фактические classified
  availability attempts. `target_unreachable`, `rate_limited`, `unclassified` и
  suppressed calls исключены. `provider_blocked` остаётся в rate, но alert-ит немедленно
  независимо от threshold.
- Automatic close использует hysteresis: `transient` rate должен быть `< 10%` за rolling
  10 минут при sample `>= 20`. Недостаточный sample и отсутствие calls recovery не
  доказывают и alert не закрывают. `provider_blocked` следует отдельным durable circuit
  recovery rules.
- Приложение не хранит `transient` alert state в PostgreSQL/Valkey: оно экспортирует
  counters через internal Prometheus-compatible `/metrics`, а open/close state и
  hysteresis вычисляет monitoring layer. Отдельные rows, domain events, audit и outbox
  отсутствуют. App restart/counter reset не считаются recovery; monitoring rate logic
  обрабатывает counter reset.
- Call-attempt telemetry использует один counter
  `trailbase_bot_provider_call_attempts_total{provider,normalized_outcome}`. Closed
  `normalized_outcome` values:
  `success|provider_blocked|target_unreachable|rate_limited|transient|unclassified`;
  `provider=telegram|max`. Method, identity, target, route, status text и error-detail
  labels и отдельный rate-limit/call-outcome counter отсутствуют. Monitoring строит
  rate/sample/alerts из этого counter.
- Cooldown gauge — `trailbase_bot_provider_cooldown_remaining_seconds{provider}`. Он
  вычисляется из durable provider `cooldown_until`, показывает remaining seconds и равен
  `0` при неактивном cooldown; единственный label — `provider=telegram|max`. Absolute
  timestamp gauge, identity/target labels и отдельная alert-state metric отсутствуют.
- Provider backlog telemetry имеет два gauges:
  `trailbase_bot_provider_queue_depth{provider}` и
  `trailbase_bot_provider_oldest_pending_age_seconds{provider}`. Depth считает все
  non-terminal delivery rows, snapshot-нутые на provider, включая отложенные cooldown,
  circuit или local throttle; oldest age — секунды от `created_at` самой старой такой
  row. Оба равны `0` при пустом backlog и имеют только `provider=telegram|max`; status,
  target и identity labels отсутствуют.
- Backlog alert опирается на oldest pending age, поскольку он отражает user-visible
  delivery delay и сопоставим между providers при разном traffic. Queue depth остаётся
  dashboard/context metric без самостоятельного fixed-threshold alert: высокая depth
  может быть нормальной при большом throughput.
- MVP backlog alert открывается, когда oldest pending age `> 15 минут` непрерывно 5 минут
  для конкретного provider. Это превышает 10-minute fresh-auth lifetime и 15-minute
  detached-unlink delivery budget и отсекает краткий retry/backoff spike. Active cooldown
  alert не подавляет: задержка остаётся user-visible; queue depth служит только context.
- Backlog alert close использует hysteresis: oldest age должен быть `< 5 минут`
  непрерывно 5 минут. Пустой backlog даёт gauge `0` и удовлетворяет условию. Metrics
  silence или scrape failure recovery не доказывают и alert не закрывают; state остаётся
  только в monitoring layer.
- Monitoring группирует одновременные `transient`, backlog, cooldown и circuit alerts по
  `provider` в одно operator notification, сохраняя отдельные alert names и conditions
  внутри группы. `provider_blocked` notification отправляется немедленно: grouping не
  ждёт связанных alerts. Identity/target labels и user payload отсутствуют.
- MVP cooldown alert открывается, когда
  `trailbase_bot_provider_cooldown_remaining_seconds{provider} > 0` непрерывно 10 минут.
  Короткий normal `Retry-After` alert не создаёт; sustained throttling создаёт отдельное
  condition, не открывающее circuit. В grouped notification backlog gauges служат
  context.
- Cooldown alert закрывается, когда gauge равен `0` непрерывно 5 минут. Новый `429`,
  продливший `cooldown_until` в этот период, сбрасывает recovery window. Metrics silence
  или scrape failure alert не закрывают; state и resolved notification принадлежат
  monitoring layer.
- Grouped provider incident отправляет один resolved notification только когда у provider
  не осталось active `transient|backlog|cooldown|circuit` conditions. Resolution
  отдельного condition при активных соседних alerts только обновляет group state без
  notification. Message содержит provider, coarse alert names и incident duration без
  identity/target/user payload.
- Severity provider conditions фиксирована: `provider_blocked`/durable circuit —
  `critical`; `transient`, backlog и cooldown — `warning`. Grouped incident наследует
  максимальную severity среди active conditions. После закрытия critical condition группа
  автоматически понижается без отдельного notification до полного resolution;
  identity/target/user labels отсутствуют.
- Периодических reminders для active grouped provider incident в MVP нет. Новый
  notification отправляется только при повышении severity группы с `warning` до
  `critical`; открытие или закрытие других conditions при неизменной severity лишь
  обновляет group state. Immediate `provider_blocked` notification остаётся явным
  исключением, один финальный resolved notification сохраняется.
- `warning` и `critical` notifications направляются в один configured operator channel с
  severity в message metadata; отдельного pager/on-call route в MVP нет. Application
  только экспортирует metrics. Destination, notification delivery и retries принадлежат
  monitoring layer и deployment configuration, а не application code.
- Configured operator channel использует operationally independent transport: alert
  delivery не зависит от TrailBase Telegram/MAX adapters, bot credentials, application
  outbox или provider circuits. Тот же monitored provider path запрещён, чтобы provider
  outage не мог скрыть собственный `critical`.
- Production deployment validation требует enabled independent operator receiver и
  валидную destination configuration; без receiver monitoring stack не считается
  production-ready. Local/dev/test могут явно отключать notifications. Application
  process readiness от внешней receiver configuration не зависит.
- Production validation после schema/required-secret checks отправляет одну deployment
  smoke notification и требует успешный transport acknowledgement. Проверка принадлежит
  monitoring/deployment layer и не создаёт application route, outbox entry или durable
  health state. Message содержит только environment и marker `monitoring_test`.
- Успешный smoke acknowledgement означает, что receiver API до timeout вернул `2xx` или
  эквивалентный provider success без error payload; human read/ack не требуется. Timeout,
  network/TLS/auth failure, non-success response или provider error проваливают production
  validation.
- Deployment smoke выполняет максимум 3 attempts, каждый с timeout 5 секунд, и backoff 1
  и 4 секунды. Retry разрешён только для timeout/network failures, `429` и `5xx`;
  auth/config errors и остальные `4xx` fail fast. После исчерпания budget production
  validation завершается ошибкой без background retries.
- Для всех attempts одного deployment smoke используется один stable idempotency key в
  receiver transport metadata. Если receiver не поддерживает idempotency, максимум три
  одинаковых `monitoring_test` сообщения допустимы; application storage и собственный
  deduplication layer не добавляются.
- Idempotency key — random UUID, созданный один раз на production validation run. Он
  повторно используется только для attempts этого run; rerun и следующий deployment
  создают новый key. Key не является secret, передаётся только в transport metadata и не
  сохраняется после завершения validation process.
- Deployment smoke validation пишет один structured final event с environment, outcome
  `success|failed`, attempt count, total duration, normalized final error class и HTTP
  status при наличии. Idempotency key, destination/URL, credentials, request headers,
  message body и raw receiver response не логируются; application audit/domain storage
  не используется.
- Закрытый smoke `error_class` vocabulary:
  `configuration|timeout|network|tls|auth|rate_limited|receiver_rejected|receiver_unavailable|provider_error|unknown`.
  При `success` поле отсутствует. HTTP `429` отображается в `rate_limited`, остальные
  `4xx` — в `receiver_rejected`, кроме auth, а `5xx` — в `receiver_unavailable`; raw
  error text в event не попадает.
- Structured event называется `monitoring_receiver_smoke_validation` и содержит
  `environment`, `outcome`, `attempt_count`, `duration_ms`, optional `error_class` и
  `http_status`. `outcome=success|failed`, `attempt_count` — integer 1–3, `duration_ms` —
  non-negative integer; timestamp и level добавляет logging framework.
- Final event имеет level `INFO` при `outcome=success` и `ERROR` при `outcome=failed`,
  поскольку failure блокирует production deployment. Отдельные per-attempt log events и
  промежуточный `WARN` перед final outcome не создаются.
