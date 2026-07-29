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
- `users`, `user_identities`
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
- `src/trailbase/auth.clj` — token.issue/verify, session cookie set/get, identity bind/link.
- Web-session/link tokens и sessions находятся в Valkey; PostgreSQL хранит users,
  messenger identities, roles и audit.
- `GET /auth` показывает confirmation; `POST /auth` атомарно расходует token и выдаёт
  `__Host-trailbase_session`, но не активирует account.

**Acceptance**:
- Первый валидированный Telegram `/start` в private one-to-one chat для неизвестной
  identity атомарно создаёт active account и identity; повторные или конкурентные
  deliveries идемпотентно resolve в тот же account.
- Созданная Telegram identity может выполнять разрешённые chat commands без web
  session.
- Telegram bot `/start` может предложить optional browser deep-link; confirmation POST
  создаёт browser session, но не является условием account/chat access.
- `/start <link-token>` от второго provider атомарно привязывает candidate identity к
  target account и не создаёт отдельный account; browser completion не требуется.
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
- Logout текущей сессии, logout-all и UI active sessions работают; sessions имеют
  sliding TTL один год.
- Email, phone, password и recovery flow отсутствуют.

**Non-goal**: фактическая загрузка и поиск треков — M03 и M06.

---

## M03 — GPX Upload + Parse

**Объём**: web/bot upload → encrypted private raw в S3 → async parse job → permanent
private draft → moderated immutable revision.

**Файлы/компоненты**:
- `src/trailbase/storage/s3.clj` — aws-simple-sign подпись + hato put/get-object + presigned URL.
- `src/trailbase/parse/gpx.clj` — hardened pull-parser → 2D MultiLineString,
  elevation profile, duration variants, metrics и warnings.
- `src/trailbase/tracks.clj` — upload flow (write S3 + insert DB), simplify-geometry (compute z11/13/15 в PostGIS-side helper или Clojure-side `simplify`).
- PostGIS-функция в `003-tracks-geom.up.sql` (migratus) —
  `populate_simplified_geometries(track_revision_id)` заполняет три revision columns.
- Activity-auto-suggestion: рудиментарный classifier показывает до трёх ranked
  вариантов с confidence; пользователь обязан выбрать activity явно.
- Htmx-форма `/tracks/new`: upload создаёт async job; polling открывает private draft
  preview; пользователь подтверждает metadata и отправляет revision на moderation.
- Bot-команда `/upload` → бот принимает file_id → backend скачивает файл через provider
  API → тот же S3+parse+DB flow → обязательные metadata и отправка на moderation
  завершаются в private one-to-one chat; web deep-link optional. Group/channel upload
  не создаёт job или draft.

**Acceptance**:
- Web и bot принимают несжатый GPX 1.0/1.1 до 10 MiB и максимум 250 000 points.
- Успешный parse создаёт private revision snapshot с 2D MultiLineString, metrics,
  user-confirmed activity и S3 references.
- `geometry_simplified_z11/13/15` populated для revision с tolerances 40/10/2 м.
- Bot `/upload` с GPX-файлом создаёт draft-track с распарсенными auto-derived; user
  подтверждает обязательные metadata и отправляет revision на moderation без web
  session.
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
- **I18n**: partial-render с языковым выбором; minimum: ru, en, simple (tags/POI names).

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
