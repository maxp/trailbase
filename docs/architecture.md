# TrailBase — Architecture

TrailBase — открытый каталог GPX-треков с отображением на карте, поиском, классификацией и газеттиром известных локаций (POI). Доступ через веб-сайт (first-class) и ботов Telegram/Max.

Масштаб: **публичный каталог сообщества** — десятки тысяч треков, тысячи пользователей, открытая загрузка с модерацией.

## Зафиксированные решения

| Домен | Решение | ADR |
|---|---|---|
| Масштаб и владение | Публичный каталог, multi-user, модерация | — |
| БД | PostgreSQL + PostGIS | A05 |
| Хранение raw | S3-совместимое (MinIO/Garage/R2) | A07 |
| Поиск | Единственный engine: PostgreSQL (tsvector+GIN мультиязычный, GIST, btree, JOIN через POI) | A05 |
| Auth | Bot-first IdP (Telegram/Max) via deep-link bridge → server-side session cookie; email/phone только recovery | A01 |
| GPX-доставка | GPX в S3 raw → парсинг → geometry в PostGIS (pre-simplified колонки z11/13/15) + auto-derived facets | A02 |
| Доставка карты | Готовые OSM raster-тайлы (без генерации тайлов) + adaptive GeoJSON по bbox + zoom-aware simplification | A02 |
| Фронтенд | htmx + alpine.js (server-rendered partials) + MapLibre GL JS как остров с bridge glue | A03 |
| Боты | Login bridge + async уведомления/модерация-инбокс, без зеркалирования CRUD-UI | A01 |
| Классификация | Иерархическая таксономия: основной тип активности (enum) + вторичные теги (moderated vocabulary) | — |
| Структурные фасеты | difficulty (по стандарту активности: SAC/MTB-Scale/3-level), season (bitmask), duration (auto из GPX или manual, `duration_source`) | A06 |
| POI / газеттир | Гибрид: самостоятельные сущности Point/Line/Area + OSM-provenance кэш, наполнение модератором/заявкой | A04 |
| Рендеринг треков на клиенте | Adaptive zoom: low-zoom кластер POI (server-side clustering в PostGIS), mid/high-zoom упрощённые полилинии | A02 |
| Backend язык | Clojure | A08 |
| Clojure-стек | reitit+ring, next.jdbc, honeysql+hugsql, Malli, Hiccup, migratus, data.xml, hato+proxy, aws-simple-sign, jsonista, telemere, deps.edn+babashka | A08 |

## Домен

### Треки
- Хранят: geometry (LineString w/ optional Z), метаданные, фасеты (difficulty/season/duration), activity_type, tags, привязку к POI (track_locations), фото.
- Auto-derived из GPX при загрузке: length, elevation gain/loss, duration (если есть `<time>`), activity-предсказание как подсказка.
- Pre-computed упрощённые геометрии: 3 уровня (z11/z13/z15) в `geometry_simplified_*`, on-the-fly fallback.
- discovery: bbox-filtered, zoom-aware simplification; NOT все треки в низких zoom'ах.

### POI (газеттир)
- Сущности: Point / LineString / Polygon (подтипом), с `osm_ref` provenance.
- Трек-связь: `track_locations(track_id, location_id, seq, distance_along)` с порядком прохождения.
- Автодетект вдоль трека при загрузке: 50 м для точек, 100 м для линий, ST_Intersects для полигонов. Read-only для загрузчика, новые POI — заявка в модерацию.
- Low-zoom: server-side clustering в PostGIS, отдаётся клиенту как ~500 кластеров.

### Поиск
- Единственный backend — PostgreSQL.
- Текст: `tsvector` мультиязычный (russian/english/simple) + GIN + триграммы для fuzzy.
- Гео: GIST-индекс, ST_Intersects/ST_DWithin.
- Фасеты: compound btree indexes.
- POI-join: JOIN через `track_locations`.
- Instant-search (<200ms), server-side aggregation по 1-2 primary фасетам (activity, difficulty).

### Auth
- Bot-first: Telegram/Max — единственный канал login. Email/phone — recovery-only.
- Bot = IdP: `/start` → short-lived token → deep-link URL → web-backed verifies → создаёт session cookie.
- Единый аккаунт с провайдерным списком (`user_identities`: login провайдеры `telegram:*`/`max:*` vs `user_recovery` для email/phone).
- Боты: login bridge + async уведомления/модерация-инбокс. Боты НЕ зеркалируют CRUD-UI.

## Стек доставок (dataflow)

```
[Telegram/Max bot]
    │ /start → token
    ▼
[deep-link URL]──▶[Web (htmx+alpine+MapLibre)]
                         │ session cookie
                         ▼
                    [Clojure backend (reitit+ring)]
                         │
            ┌────────────┼────────────┐
            ▼            ▼            ▼
       [PostGIS]      [S3]       [Bot API]
       geometry/      raw GPX    notifications
       metadata       photos
```

## Срезы (tracer bullets) — см. ROADMAP

Декомпозиция на вертикально доставляемые срезы — см. `docs/ROADMAP.md` (после ADR-серии).