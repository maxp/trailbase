# TrailBase — Architecture

TrailBase — открытый каталог GPX-треков с отображением на карте, поиском, классификацией и газеттиром известных локаций (POI). Доступ через веб-сайт (first-class) и ботов Telegram/Max.

Масштаб: **публичный каталог сообщества** — десятки тысяч треков, тысячи пользователей, открытая загрузка с модерацией.

Практические инварианты реализации, принятые после ADR-серии, собраны в
[Implementation Contract](IMPLEMENTATION-CONTRACT.md). При расхождении краткого
обзора или roadmap с этим контрактом действует контракт.

## Зафиксированные решения

| Домен | Решение | ADR |
|---|---|---|
| Масштаб и владение | Публичный каталог, multi-user, модерация | — |
| БД | PostgreSQL + PostGIS | A05 |
| Хранение raw | Private S3-compatible storage; encrypted raw и sanitized export | A07 |
| Поиск | Единственный engine: PostgreSQL (tsvector+GIN мультиязычный, GIST, btree, JOIN через POI) | A05 |
| Auth | Telegram/Max identity достаточна для chat access; optional one-time token → Valkey-backed web session; email/phone отсутствуют | A01 |
| GPX-доставка | Encrypted raw в S3 → async parse → 2D MultiLineString revision в PostGIS + sanitized export | A02 |
| Доставка карты | Готовые OSM raster-тайлы (без генерации тайлов) + adaptive GeoJSON по bbox + zoom-aware simplification | A02 |
| Фронтенд | htmx + alpine.js (server-rendered partials) + MapLibre GL JS как остров с bridge glue | A03 |
| Боты | First-class chat UI: identity, upload, search и уведомления поверх общих domain services | A01 |
| Классификация | Иерархическая таксономия: основной тип активности (enum) + вторичные теги (moderated vocabulary) | — |
| Структурные фасеты | difficulty (по стандарту активности: SAC/MTB-Scale/3-level), season (bitmask), duration (auto из GPX или manual, `duration_source`) | A06 |
| POI / газеттир | Гибрид: самостоятельные сущности Point/Line/Area + OSM-provenance кэш, наполнение модератором/заявкой | A04 |
| Рендеринг треков на клиенте | Adaptive zoom: low-zoom кластер POI (server-side clustering в PostGIS), mid/high-zoom упрощённые полилинии | A02 |
| Backend язык | Clojure | A08 |
| Clojure-стек | reitit+ring, next.jdbc, honeysql+hugsql, Malli, Hiccup, migratus, data.xml, hato+proxy, aws-simple-sign, jsonista, telemere, deps.edn+babashka | A08 |

## Домен

### Треки
- `tracks` хранит stable identity и `current_revision_id`; versioned geometry,
  метаданные, фасеты и tags находятся в immutable `track_revisions`.
- Canonical geometry — 2D MultiLineString. Elevation profile хранится отдельно.
- Auto-derived из GPX при загрузке: length, elevation gain/loss, duration (если есть `<time>`), activity-предсказание как подсказка.
- Pre-computed упрощённые геометрии: 3 уровня (z11/z13/z15) в `geometry_simplified_*`, on-the-fly fallback.
- discovery: bbox-filtered, zoom-aware simplification; NOT все треки в низких zoom'ах.

### POI (газеттир)
- `locations` имеет stable identity и immutable revisions. Geometry kind
  (`point/line/area`) отделён от semantic category.
- POI links — модерируемые annotations конкретной geometry revision; одна связь может
  иметь несколько ordered occurrences.
- High-confidence autodetect approved locations публикуется автоматически, остальные
  совпадения идут в moderation.
- Low-zoom использует deterministic server-side hex-grid clustering в PostGIS.

### Поиск
- Единственный backend — PostgreSQL.
- Текст: `tsvector` мультиязычный (russian/english/simple) + GIN + триграммы для fuzzy.
- Гео: GIST-индекс, ST_Intersects/ST_DWithin.
- Фасеты: compound btree indexes.
- POI-join: JOIN через approved revision-location annotations.
- Instant-search (<200ms target), все facet counts считаются server-side с
  disjunctive semantics.

### Auth
- Telegram/Max — единственные identity providers; email/phone/recovery отсутствуют.
- Валидированная messenger identity достаточна для account и разрешённых chat
  operations; browser activation не требуется.
- Первый валидированный `/start` неизвестной identity в private one-to-one chat
  атомарно создаёт active account и provider identity; повторная доставка идемпотентно
  использует тот же account. Identity и token flows в группах/каналах запрещены.
- Optional web entry: bot выдаёт 10-minute Valkey token → safe GET/POST confirmation →
  Valkey session cookie.
- Единый аккаунт с explicit provider linking: `/start <link-token>` второго provider
  добавляет identity к target account без нового account и browser completion;
  автоматического account merge нет.
- Session cookie содержит 128-bit opaque token; sliding TTL — один год.
- Telegram/Max chats поддерживают upload и search без web session, используя те же
  domain services, permission checks и async pipelines, что web/API.
- Upload выполняется только в private chat. Group/channel `/search` работает без
  account state и видит только public published catalog.
- Interactive search callback несёт только opaque ID; 15-минутные binding/query/cursor
  records находятся в Valkey и не восстанавливаются после потери.

## Стек доставок (dataflow)

```
[Telegram/Max chat]──────── identity/upload/search ────────┐
       │ optional web-session token                       │
       ▼                                                  ▼
[deep-link URL]──▶[Web (htmx+alpine+MapLibre)]──▶[Clojure backend]
                                                        │
                                   ┌────────────┬────────┼────────┐
                                   ▼            ▼        ▼        ▼
                              [PostGIS]      [S3]    [Valkey] [Bot API]
                              revisions/ encrypted  sessions/ responses/
                              geometry   raw/export streams    notifications
```

## Срезы (tracer bullets) — см. ROADMAP

Декомпозиция на вертикально доставляемые срезы — см. [Roadmap](ROADMAP.md).
