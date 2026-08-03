# Контракт: monitoring и deployment control

Часть [Implementation Contract](../../IMPLEMENTATION-CONTRACT.md). При противоречии с другими документами действует этот контракт.

- Receiver smoke запускается после применения и успешной schema validation monitoring
  configuration, но до application traffic cutover. Smoke failure abort-ит deployment и
  оставляет предыдущую release обслуживать traffic; при первом deployment traffic не
  включается до успешного smoke.
- Failed smoke автоматически восстанавливает last-known-good monitoring configuration и
  требует успешный reload. При first deployment без предыдущей config candidate
  отключается, monitoring stack остаётся non-production-ready и traffic не включается.
  Application release не откатывается, поскольку cutover ещё не происходил.
- Last-known-good — immutable versioned artifact последнего deployment с успешным receiver
  smoke. Deployment system обновляет pointer только после success. Artifact содержит
  monitoring config и версии/ссылки на secrets, но не secret values; текущий live
  filesystem не является source of truth.
- Production deployments сериализуются exclusive lock-ом на environment от candidate
  apply до rollback либо successful pointer update и traffic cutover. Last-known-good
  pointer обновляется atomic compare-and-swap из ожидаемой previous version; stale runner
  завершается failure и не выполняет rollback/cutover поверх нового deployment.
- Exclusive lock реализует native concurrency group deployment orchestrator-а с key
  `trailbase-deploy-${environment}` и без cancel-in-progress: новый deployment ждёт
  текущий, а lock освобождается orchestrator-ом при finish/crash job. PostgreSQL, Valkey и
  custom lock service не используются; pointer CAS остаётся обязательной второй защитой.
- Для environment существует один active и только latest pending production deployment.
  Новый pending supersede-ит более старый до candidate apply; active job не отменяется.
  Superseded job завершается `cancelled|skipped`, не отправляет smoke notification и не
  пишет `monitoring_receiver_smoke_validation`, поскольку validation не запускалась.
- Manual deployment и emergency rollback используют ту же
  `trailbase-deploy-${environment}` concurrency group и pointer CAS: manual bypass для
  monitoring config, last-known-good pointer или traffic cutover отсутствует. Emergency
  rollback становится latest pending; parallel mutation запрещена.
- Explicit cancellation active production deployment обрабатывает cancellation handler,
  пока environment lock удерживается. До candidate apply cleanup не нужен; после apply,
  но до pointer update/traffic cutover, handler восстанавливает last-known-good и требует
  successful reload. Pointer update + traffic cutover — non-cancellable critical section.
  После неё cancellation не откатывает release; emergency rollback запускается новым
  queued deployment.
- Если cancellation cleanup не смог восстановить или reload last-known-good monitoring
  config, deployment завершается failed без traffic cutover. Каждый следующий queued job
  до любых mutations выполняет mandatory reconciliation preflight: live monitoring config
  artifact/digest должен совпасть с last-known-good pointer. Mismatch завершает job с
  `monitoring_state_diverged`; candidate apply и cutover запрещены до manual reconciliation
  через ту же concurrency group. Application PostgreSQL/Valkey state для этого не
  добавляется.
- При существующем last-known-good pointer manual reconciliation выполняет отдельный
  operator-triggered `reconcile-monitoring` job в concurrency group
  `trailbase-deploy-${environment}`. Job не позволяет выбрать произвольную версию: он
  повторно применяет указанный pointer-ом immutable artifact и его secret
  references/versions, затем выполняет schema validation, reload, проверку live digest и
  receiver smoke. Pointer, application release и traffic cutover не меняются; отдельный
  durable reconciliation flag не добавляется.
- При first deployment без last-known-good pointer тот же `reconcile-monitoring` job в
  concurrency group удаляет или отключает частично применённый candidate и проверяет
  отсутствие active production monitoring config. Monitoring остаётся
  non-production-ready, traffic выключен, synthetic last-known-good artifact/pointer не
  создаётся. Следующий обычный deployment начинает с явного no-pointer state и обязан
  пройти apply, schema validation, reload и receiver smoke до создания первого pointer и
  traffic cutover.
- Adopt-live/force режим при `monitoring_state_diverged` запрещён. Last-known-good pointer
  может указывать только на immutable versioned deployment artifact, прошедший полный
  schema validation, reload и receiver smoke; `reconcile-monitoring` не принимает live
  filesystem как source of truth и не двигает pointer. Другой config вводится новым
  versioned deployment или emergency rollback через ту же concurrency group и queue после
  reconciliation; force bypass отсутствует.
- Если `reconcile-monitoring` не может получить точную secret version, указанную
  last-known-good artifact, он завершается fail closed с
  `monitoring_secret_version_unavailable`; fallback на latest secret value запрещён.
  Secret backend обязан сохранять version, на которую ссылается current last-known-good
  pointer, пока pointer не продвинут и artifact нужен active deployment/reconciliation.
  Operator восстанавливает точную version в secret backend и повторно запускает
  `reconcile-monitoring`; adopt-live, candidate apply и traffic cutover остаются
  запрещены.
- Rollback set каждого environment содержит current last-known-good и ровно один previous
  successful monitoring artifact; exact secret versions обоих artifacts pinned. GC
  удаляемого artifact и его versions разрешён только после successful pointer advance и
  завершения всех jobs, которые ещё на него ссылаются. Direct emergency rollback выбирает
  только previous artifact. Более глубокое восстановление выполняется новым versioned
  deployment из source; отдельного time-based retention TTL нет.
- Direct emergency rollback не создаёт новый monitoring artifact или byte-copy. Новый
  orchestrator run с новым smoke idempotency UUID применяет exact immutable previous
  artifact и его pinned secret versions, выполняет schema validation, reload, live-digest
  check и receiver smoke. Только после success pointer CAS переключает current на previous
  и выполняется traffic cutover; роли artifacts меняются местами. При failure до commit
  pointer остаётся прежним и current config восстанавливается.
- Если traffic cutover fail-ится после successful pointer CAS, deployment под тем же
  environment lock выполняет compensating rollback: сохраняет или возвращает traffic на
  prior release, CAS-ом возвращает pointer на former current artifact, восстанавливает его
  monitoring config и требует successful reload. Полностью успешная compensation завершает
  deployment failed, но оставляет environment consistent. Failure любого compensation
  step завершает job с `deployment_commit_incomplete`; дальнейшие automatic mutations
  запрещены до operator reconciliation через ту же concurrency group.
- Durable environment deployment-control object под тем же CAS, что current/previous
  pointers, хранит state `ready|migrating|schema_ahead|commit_incomplete`, expected active
  release ID и state-specific source/target migration digests и job ID без secrets. Каждый
  deploy/rollback preflight сверяет state, live traffic target и monitoring digest.
  Guard снимает только operator-triggered `reconcile-deployment` job в той же concurrency
  group: он возвращает traffic, pointer и monitoring config к former current, выполняет
  reload, digest check и receiver smoke, затем CAS-ом ставит `ready`. Application Valkey
  state не добавляется; PostgreSQL наблюдается только через exact applied IDs и
  migration-head digest, определённые ниже.
- Каждый job под environment concurrency group начинает с derived consistency preflight:
  читает deployment-control object и фактические live traffic target/monitoring digest.
  Любое несовпадение при state `ready` fail closed CAS-ом переводит object в
  `commit_incomplete` с coarse reason `preflight_mismatch` и expected IDs без secrets,
  затем job abort-ится без mutations. При CAS conflict job перечитывает object и всё равно
  abort-ится; продолжить может только `reconcile-deployment`, automatic compensation
  обычным deploy запрещена.
- Если consistency preflight не может получить live traffic target или monitoring digest,
  job завершается fail closed с `deployment_state_unobservable` до любых mutations.
  Отсутствие observation не доказывает mismatch, поэтому `ready` автоматически не
  переводится в `commit_incomplete`; уже установленный guard сохраняется. Создаётся
  operator alert, retry выполняется только новым queued run после восстановления
  control-plane read. `reconcile-deployment` также не может поставить `ready` без полного
  traffic-target и digest observation.
- Deployment использует два consistency gates. Initial preflight до mutations подтверждает
  current pointer/digest и active traffic release. Commit gate после candidate smoke и
  непосредственно перед non-cancellable pointer/cutover section повторно проверяет
  неизменную CAS generation, current/previous IDs, prior traffic target и expected candidate
  live digest. Любой drift запрещает commit и запускает restore current config. Successful
  restore завершает job с `deployment_state_changed`; failed restore переводит environment
  в `commit_incomplete`.
- Routine write credentials к live traffic target, active monitoring config и
  deployment-control object получает только deployment automation service identity; все
  writes выполняются под `trailbase-deploy-${environment}` concurrency group. Human
  operator может только trigger-ить deploy/rollback/reconcile jobs, direct write запрещён.
  Audited break-glass workflow также входит в ту же group, первым CAS-ом ставит
  `commit_incomplete`, пишет только actor/reason и expected IDs без secrets, а вернуть
  `ready` может лишь через `reconcile-deployment`. Неучтённые out-of-band edits не
  поддерживаются.
- Production break-glass получает только short-lived per-run credential через orchestrator
  OIDC federation в dedicated environment-scoped infrastructure role. Static/shared keys и
  выдача raw credential человеку запрещены. Production protected-environment approval
  обязателен до issuance; role может менять только traffic target, monitoring config и
  deployment-control object выбранного environment. Provider audit получает actor, reason
  и run ID без credential/token values; credential автоматически истекает после run.
- Initiator production break-glass не может approve собственный protected-environment gate.
  Требуется ровно один authorized reviewer, отличный от triggering actor: минимальный
  two-person control. Reason заполняется до approval; reviewer видит target environment и
  intended operation. Без независимого reviewer workflow остаётся blocked; self-approval
  fallback и обход через static credential отсутствуют. Approval actor, timestamp и run ID
  сохраняет orchestrator audit; политика обычных deploy/rollback определяется отдельно.
- Отдельный distinct-reviewer approval для ordinary production deploy, direct emergency
  rollback и `reconcile-deployment` не требуется. Ordinary deploy допускается автоматически
  только из protected main после required CI; merge approval уже является change review.
  Direct rollback и reconciliation может trigger-ить один authorized production operator
  без второго approval, поскольку jobs ограничены retained artifacts, CAS и consistency
  gates. Все runs audited и проходят общую concurrency group. Distinct reviewer обязателен
  только для arbitrary break-glass mutation.
- Immutable production release identity создаёт required CI: он один раз собирает
  application release artifact и monitoring artifact из exact protected-main commit.
  Deploy получает commit SHA и content digests, проверяет native artifact
  provenance/attestation и использует artifacts только по digest; rebuild и resolution
  mutable branch/tag запрещены. Deployment-control object сохраняет commit SHA, application
  artifact digest и monitoring artifact digest без secrets. Missing artifact или invalid
  provenance fail-ит job с `deployment_artifact_untrusted` до любых mutations.
- Rollback set хранит полную согласованную release pair, не только monitoring artifact.
  Каждый current/previous slot указывает на один immutable release manifest с commit SHA,
  application artifact digest, monitoring artifact digest и exact secret
  references/versions. Direct rollback выбирает manifest целиком; смешивать application из
  одного commit с monitoring из другого запрещено. GC удаляет manifest, оба artifacts и
  secret-version pins как одну retention unit только после pointer advance и завершения
  всех referencing jobs.
- Release manifest содержит exact migration-head digest. Direct rollback preflight
  разрешает переключение на previous application только когда её migration-head digest
  равен текущему production DB head. При различии job завершается
  `rollback_schema_incompatible` до mutations; automatic down migrations и попытка
  запустить старое application поверх новой schema запрещены. Recovery выполняется
  forward-fix release из protected main, совместимым с уже применённой schema.
- Required CI формирует canonical ordered migration manifest из пар `migration_id +
  SHA-256(up.sql)` для всех migrations release; migration-head digest равен SHA-256
  canonical manifest bytes, а сам manifest защищён общей immutable artifact provenance.
  Deploy сверяет exact ordered applied IDs из Migratus table с manifest; проверка только
  максимального migration ID и runtime schema introspection запрещены. Deployment-control
  сохраняет resulting digest без SQL text.
- Migration preflight требует, чтобы production applied IDs были exact prefix target
  manifest, а сохранённый production digest совпадал с digest этого prefix. Gap, reorder
  или drift завершают job с `deployment_schema_drift` до mutations. One-shot Migratus под
  exclusive DB advisory lock применяет только suffix migrations, пока current application
  продолжает обслуживать traffic; поэтому этот шаг допускает только additive expand
  migrations. После success deployment-control фиксирует новый DB digest, deploy
  переключает application и выполняет обычные gates. Destructive contract migration идёт
  отдельным позднейшим release; несовместимый previous image после неё не запускается,
  recovery остаётся forward-fix.
- После migration success и до application cutover deployment-control CAS-ом переходит в
  `schema_ahead` с target migration digest. Current application остаётся обслуживать traffic
  на additive schema. Failure или cancellation candidate не запускает down migration и не
  меняет release pointers: сохраняет `schema_ahead`, поднимает alert и разрешает через общую
  concurrency group только forward deploy, чей manifest принимает текущий DB head как exact
  prefix. Successful candidate cutover и gates одним CAS обновляют current/previous pointers
  и возвращают `ready`; rollback/reconcile к schema-incompatible release в `schema_ahead`
  запрещены.
- До первой DB mutation deployment-control CAS-ом переходит в durable `migrating` с source
  digest, target digest и job ID. Каждая production migration выполняется transactionally;
  Migratus `:disable-transaction` запрещён. После crash следующий job под общей concurrency
  group сверяет exact applied IDs: source head допускает safe restart, strict intermediate
  prefix — resume оставшегося suffix, target head — CAS в `schema_ahead`. Любой non-prefix
  или ambiguous state переводится в `commit_incomplete` с reason
  `migration_state_ambiguous` без automatic mutations.
- Non-transactional и долгие DB operations не входят в Migratus migrations; migration
  manifest содержит только короткие transactional schema changes. `CREATE INDEX
  CONCURRENTLY`, chunked backfill и долгие validations выполняются отдельными versioned
  DB-operation jobs из attested release artifact с idempotent/resumable steps, durable
  checkpoints, progress/alerting и dedicated DB advisory lock под общей environment
  concurrency group. Expand application можно выпустить до завершения backfill, но release,
  использующий новый index/data invariant, и любая contract migration блокируются explicit
  completion gate.
- Production PostgreSQL хранит durable ledger `deployment_db_operations`: immutable
  `operation_id`, attested `artifact_digest`, closed state
  `pending|running|succeeded|failed`, `checkpoint jsonb` и timestamps. Каждый backfill chunk
  изменяет domain rows и checkpoint одной transaction. `succeeded` записывается только после
  operation-specific validator; concurrent-index resume сверяет exact expected catalog
  definition. Release manifest перечисляет required `operation_id + artifact_digest`, а
  deploy completion gate принимает только exact `succeeded` matches. SQL text, secrets и raw
  error payload в ledger не сохраняются.
- DB-operation claim выполняется atomic conditional update: записывает `lease_owner`,
  `lease_expires_at`, `heartbeat_at` и увеличивает `attempt_count`. После lease expiry новый
  job с тем же artifact digest reclaim-ит `running` и resume-ит durable checkpoint. `failed`
  retry-ится только explicit operator-triggered job с тем же digest; checkpoint сохраняется.
  `succeeded` immutable, повторный запуск является validated no-op. Тот же `operation_id` с
  другим digest fail closed как `db_operation_artifact_mismatch` и требует нового versioned
  ID; force reset/overwrite отсутствует.
- DB-operation cancellation cooperative: durable `cancel_requested_at` проверяется перед
  каждым chunk и validator, active statement получает bounded driver cancellation. Backfill
  transaction откатывает только current chunk; committed checkpoint сохраняется, row
  становится `failed` с safe code `cancelled`, lease очищается. После concurrent-index cancel
  validator классифицирует catalog: exact valid index даёт `succeeded`; exact expected invalid
  index удаляется controlled `DROP INDEX CONCURRENTLY`, затем state становится `failed`;
  unexpected definition даёт `failed` с `db_operation_state_ambiguous` без automatic cleanup.
  Отдельный `cancelled` state не добавляется.
- DB-operation удерживает `trailbase-deploy-${environment}` concurrency group от successful
  claim до terminal validation/cleanup. Все deploy, rollback и reconcile jobs ждут эту group;
  chunk-level release и parallel DB-operation отсутствуют. Для emergency operator отменяет
  active workflow: cancellation handler выставляет `cancel_requested_at`, завершает принятую
  классификацию и освобождает lease/group, после чего queued rollback может стартовать.
  Dedicated DB advisory lock остаётся secondary guard. Смена application/schema assumptions
  посреди backfill запрещена; production mutations образуют одну последовательность.
- Cancellation handler получает пять минут grace period. Если terminal classification не
  получена, workflow завершается failed, оставляет row `running` с `cancel_requested_at` и
  поднимает alert; lease не force-expire-ится. Каждый queued deploy/rollback preflight при
  non-terminal row или недоступном DB advisory lock завершается
  `db_operation_unresolved` без mutations. После lease expiry operator запускает
  `reconcile-db-operation` в той же group: job reclaim-ит exact artifact, валидирует
  фактическое DB state и завершает cancellation. Только terminal row снимает gate; force
  kill/ignore отсутствует.
- `deployment_db_operations` rows и checkpoints в MVP не удаляются и не переписываются:
  это малый control/audit history. Attested operation artifact pin-ится, пока row имеет state
  `pending|running|failed`, на него ссылается current/previous release manifest или active
  job. После `succeeded`, отсутствия retained release/job references и pointer advance
  artifact удаляется общей release retention unit; отдельного time TTL нет. Failed artifact
  остаётся pinned до successful retry; manual abandon/GC path не добавляется.
- DB-operation observability использует только low-cardinality metrics: active gauge по
  `environment,kind,state`, attempts counter по `environment,kind,outcome,error_code`,
  duration histogram и checkpoint-age gauge. `operation_id`, artifact digest и lease owner в
  metric labels запрещены. Каждый attempt пишет один final structured event
  `db_operation_attempt_finished` с safe `operation_id`, kind, attempt, outcome, duration,
  processed count и allowlisted error code, без SQL/checkpoint/raw error. Alerts обязательны
  для failed/unresolved outcome, expired lease и non-terminal operation, блокирующей
  deployment; per-chunk log events отсутствуют.
