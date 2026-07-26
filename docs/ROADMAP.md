# TrailBase — Roadmap (vertical slices)

Декомпозиция дизайна (см. `architecture.md` + `docs/adr/`) на независимо доставляемые вертикальные срезы (tracer bullets). Каждый срез — **runnable end-to-end**, observable через конкретное поведение, а не горизонтальный слой.

M01 — prerequisite всех остальных. M02 → M03 → M04 — backbone. M05/M06/M07/M08 — independent extensions поверх backbone (могут разрабатываться параллельно после M04).

```
M01 Foundation
   │
   ▼
M02 Auth ──▶ M03 Upload ──▶ M04 Catalog Render
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

**Core-таблицы (M01 заложить миграциями; дополняются в M02/M03/M05)**:
- `users`, `user_identities`, `user_recovery`
- `tracks` (стартовая форма: id, owner_id, name, description, geometry, s3_uri, length_m, ele_gain_m, ele_loss_m, duration_s, duration_source, activity_type, difficulty, season, created_at)
- `geometry_simplified_z11/13/15` колонки (added в миграции, populated в M03)
- `tags`, `track_tags` (data tables populated в M07)
- `locations`, `track_locations` (schema создается в M05)
- Session storage (ring in-memory или `ring-session` table).

**Acceptance**:
- `bb migrate` создаёт PostGIS-расширение и все core-таблицы.
- `bb run-dev` поднимает HTTP-сервер; `GET /health` → 200 JSON.
- `GET /` рендерит Hiccup-страницу с layoutом; partial добавляется через `hx-get`.
- next.jdbc datasource доступен в REPL; hugsql queries загружаются.
- Перед этим: `docker-compose.yml` (PostgreSQL+PostGIS, MinIO) поднимается с одной командой.

**Non-goal**: auth, реальные треки, бот.

---

## M02 — Bot-first Auth

**Объём**: Telegram/Max bot `/start` → deep-link token → web `/auth?token=` → session cookie; единый аккаунт с провайдерами.

**Файлы/компоненты**:
- `src/trailbase/bot/telegram.clj` — raw Bot API через hato (getUpdates/setWebhook/sendMessage); polling или webhook-mode.
- `src/trailbase/bot/max.clj` — raw Bot API (Max/NetM), аналогично Telegram.
- `src/trailbase/bot/core.clj` — общая логика: бот = IdP, выдаёт одноразовый short-lived token, дип-link URL.
- `src/trailbase/auth.clj` — token.issue/verify, session cookie set/get, identity bind/link.
- Миграции: `002-auth-token.up.sql` (table `auth_tokens(token PK, identity_ref, expires_at, used_at)`).
- Htmx-endpoint `/auth?token=...` → верифицируется, создаётся user если первый вход, привязка identity, set-cookie, redirect на `/`.
- Recovery flow placeholder (email/phone) — schema есть, flow в M02 не реализуется (only schema).

**Acceptance**:
- Telegram bot `/start` → user получает deep-link в чат.
- Клик по ссылке в браузере → backend верифицирует token → создаётся user (`users` + `user_identities` row) → user залогинен (session cookie виден в DevTools), redirect на `/` с приветствием.
- Повторный `/start` в Telegram или Max под тем же bot-identity подхватывает тот же `user_id` (если identity уже есть) либо создаёт новый `user_identities` для (= метод link/ing на этом уровне не в MVP).
- Max bot — тот же flow, тот же `/auth` endpoint, identity `max:*`.
- Session cookie проверяется middleware; запрос на `/` без cookie редиректит на бота (для MVP поток "no-cookie → prompt bot login" достаточно).
- Logout-эндпоинт закрывает session.

**Non-goal**: web-форма загрузки треков, восстановление аккаунта через email (schema есть, UI нет).

---

## M03 — GPX Upload + Parse

**Объём**: загрузка raw GPX (из бота и/или веб-формы) → S3 → parse → geometry + auto-derived в PostGIS.

**Файлы/компоненты**:
- `src/trailbase/storage/s3.clj` — aws-simple-sign подпись + hato put/get-object + presigned URL.
- `src/trailbase/parse/gpx.clj` — `data.xml` pull-парсер → `{:geometry ..., :length_m, :ele_gain_m, :ele_loss_m, :duration_s, :min_ts?, :max_ts?, :points_count}`.
- `src/trailbase/tracks.clj` — upload flow (write S3 + insert DB), simplify-geometry (compute z11/13/15 в PostGIS-side helper или Clojure-side `simplify`).
- PostGIS-функция в `003-tracks-geom.up.sql` (migratus) — `populate_simplified_geometries(track_id)` заполняет 3 колонки через `ST_SimplifyPreserveTopology`.
- Activity-auto-suggestion: рудиментарный classifier на (avg_speed, ele_gain/length ratio) → hедливый ranking activity_type с confidence; предлагается как default в форме.
- Htmx-форма `/tracks/new`: выбор GPX → submit (multipart) → partial показывает parsed summary (map, length, ele, duration, suggested activity) → user корректирует activity/name/description → final POST `/tracks`.
- Bot-команда `/upload` → бот принимает file_id → backend скачивает с Telegram file proxy → тот же S3+parse+DB flow → deep-link в web-форму для финализации метаданных.

**Acceptance**:
- Веб-форма загружает GPX 1-10 MB; показывает preview: отрисованная LineString (через <svg> или быстро любую визуализацию), length/ele/duration/suggested activity.
- Подача финальной формы — row в `tracks` с populated `geometry`, `length_m`, `ele_gain_m`, `ele_loss_m`, `duration_s` (если GPX had `<time>`), `duration_source`, `activity_type` (user-confirmed), `s3_uri`.
- `geometry_simplified_z11/13/15` колонки populated post insert.
- Bot `/upload` с GPX-файлом создаёт draft-track с распарсенными auto-derived, user получает deep-link на финализацию в вебе.
- Re-parse endpoint (admin/dev): `POST /tracks/:id/reparse` перечитывает s3_uri и обновляет PostGIS-geometry (полезно для improved extractor later).

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
- При pan/zoom клиент: (а) для z≤13: fetch `/api/locations/clusters.geojson` (~500 кластеров, mark placeholders) — или tracks-as-centroids fallback, если M05 не готов; (б) для z14-15: fetch `/api/tracks.geojson?bbox=&zoom=14/15` → упрощённые LineStrings, WebGL-рендер.
- High-zoom (≥16): клик по треку → `setData` одной track's full geometry, открывается детальная partial.
- Hover трека в sidebar → полилиния на карте стилизована (highlight).
- Performance: 5000 треков в тестовой выборке — pan/zoom без lag (>30 FPS) в Chrome.

**Non-goal**: POI синтаксис (M05), real search (M06), классификация (M07).

---

## M05 — POI Gazetteer

**Объём**: каталог(locations Point/Line/Polygon) + track_locations link + autodetect + OSM-import + cluster endpoint.

**Файлы/компоненты**:
- Миграции `005-poi.up.sql`: `locations` (Geometry(4326, Geometry), type enum `point/line/area`, osm_type, osm_id, name, meta jsonb), `track_locations(track_id, location_id, seq int, distance_along_m, PRIMARY KEY(track_id, location_id))`.
- `src/trailbase/locations.clj` — CRUD для модератора, list/get, autodetect-along-track (honeysql with `ST_DWithin` / `ST_Intersects` radius-aware), cluster endpoint.
- ОSM-import pipeline: `src/trailbase/import/osm.clj` — Overpass API fetch by osm_id/type, кэш name/geometry в `locations`, provenance record.
- Autodetect: after track parse (M03 hook) — backend queries nearby POI with adaptive radius, INSERT into `track_locations` pending_state? — MVP: auto-bind track_locations with `confidence`, помечается как «read-only suggestion» для загрузчика; заявка нового POI — POST `/locations/suggest` → модерация queue (moderation table created в M05 minimum).
- Low-zoom cluster endpoint (M04 заглушка оживляется): `/api/locations/clusters.geojson?bbox=&zoom=` → ST_ClusterDBSCAN или hex-binning.
- Bot: загрузчик видит auto-detected POI как inline keyboard ("绑定 к X? да/нет / предложить новый"). Реальные правки — через модерацию.
- Мoderator UI: `/moderation/locations` — список заявок, approve → creates location → re-runs autodetect binding.

**Acceptance**:
- После загрузки трека backend autodetect-ит POI вдоль трека (radius-aware), показывает их в загрузчик-форме.
- Загрузчик может предложить новое POI через форму (имя, тип, координаты или клик на карте) — заявка в модерацию.
- Модератор одобряет заявку → POI создаётся → привязка track_locations обновляется.
- `/api/locations/clusters.geojson?zoom=10` возвращает ≤500 кластер-точек для bbox.
- Поиск «треки через локацию X» работает: `/locations/:id/tracks` список треков по `track_locations`.
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
- `src/trailbase/search.clj` — honeysql composable query builder: combine text (tsvector OR/AND `pg_trgm` word_similarity), geo bbox (`ST_Intersects`), фасеты (IN, range), POI-join (`JOIN track_locations`).
- Endpoint `/api/search?q=&bbox=&activity=&difficulty=&season=&duration_min=&duration_max=&location_id=&page=` → пагинированный результат (list partial) + server-side aggregation partial (activity counts, difficulty distribution).
- Instant-search: `hx-get` on `keyup changed delay:300ms` → htmx partial swap `#results` и `#facets`.
- Location-constraint: `location_id=N` → JOIN `track_locations` → треки через локацию N.
- Geography-aware: bbox в segmented radio — server postgis geometry comparison.

**Acceptance**:
- Free-text поиск по track name/description: tsvector + триграмы, опечатки 1-2 символа учитываются.
- Фильтр `activity=hike`, `difficulty=T3`, `season=winter`, `duration_min=3600 & duration_max=21600` (1-6h) — composable AND.
- Комбинированный: текст + фасеты + bbox одновременно — один SQL-запрос через CTE.
- POI-filter: `location_id=5` → список треков через локацию 5.
- Instant-search: ввод 3+ символов → partial swap без перезагрузки страницы; сервер откликивается <200ms на индексированных данных.
- `#facets` обновляется server-side aggregation (activity counts, difficulty distribution) при каждом изменении фильтра.
- Мультиязычность: `description` jsonb с языковыми ветвями; tsvector собирается из всех языков, ranking по combined.

**Non-goal**: external search engine (Meilisearch), semantic search (pgvector).

---

## M07 — Classification & Moderation

**Объём**: tag dictionary + очереди заявок тегов; difficulty/season/duration facet wiring; модераторский API.

**Файлы/компоненты**:
- Миграции `007-tags.up.sql`:
  - `tags(id, key, label_i18n jsonb, parent_id, status enum approved/pending/rejected, created_by, approved_by)`.
  - `track_tags(track_id, tag_id, source enum user/moderator/derived)` (UNIQUE track_id+tag_id).
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
- Elevation profile chart: raw GPX-parse stores `points` query-optimized или pre-computed `elevation_profile` jsonb column at upload (M03 hook). Client → alpine + d3/simple-svg renderer → пан-by-brush ссылается на track detail.
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
- **bot module** (raw Bot API over hato): телеграм в M02, Max в M02+; Max API может отстоязить CFL позже, но конец M02 — работающий login через оба.
- **tests**: по одному integration test на каждый срез (smoke-level: upload fixture GPX, render map page, search returns expected track). Не TDD по слоям, smoke на end-to-end.
- **I18n**: partial-render с языковым выбором; minimum: ru, en, simple (tags/POI names).

## Orders & Dependencies

- M01 — prerequisite всех.
- M02 — после M01 (moderate подмамины — bot, auth).
- M03 — после M02 (upload требует login).
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