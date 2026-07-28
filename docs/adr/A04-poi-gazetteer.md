# ADR A04 — POI / газеттир: гибрид самостоятельных сущностей + OSM-provenance

**Status**: Accepted
**Date**: 2026-07-25

**Уточнение 2026-07-28:** locations и track content versioned; POI links являются
отдельно модерируемыми revision annotations с multiple occurrences. Geometry kind
отделён от semantic category. Полный контракт:
[Implementation Contract](../IMPLEMENTATION-CONTRACT.md#16-poigazetteer).

## Контекст

TrailBase каталогирует треки. Пользователь запросил пометки **известных локаций, по которым проходит трек**. Локация — это не «тег трека», а самостоятельная сущность газеттира (горы/перевалы/приюты/водопады/деревни/заповедники/хребты), через которую проходят многие треки. Требуется модель, позволяющая (а) поиск «треки через локацию X»; (б) навигацию каталога от газеттира к трекам; (в) масштабируемое autодетект-наполнение без ручного ввода десятков тысяч POI модератором.

## Решение

1. **Сущности Point / LineString / Polygon** (подтипом в `locations`):
   - точечные: вершины, перевалы, приюты, водопады, trailheads, водные источники;
   - линейные: хребты, river reaches, ridgelines;
   - полигональные: деревни, заповедники, охраняемые зоны.
   Геометрия хранится как `geometry(Geometry, 4326)` PostGIS.
2. **Гибрид self-contained каталога + OSM-provenance кэша:**
   - Модератор может создать локацию с указанием `osm_ref` (тип+id).
   - Система фетчит OSM-объект, кэширует `name`/`geometry`, создаёт нашу локацию с provenance.
   - Records живут собственной жизнью; при рассинхроне модератор видит diff и может обновить из OSM.
3. **Revision-location annotation**: logical link относится к geometry revision и
   имеет 0..n ordered occurrences с cumulative и segment-local distance.
4. **Автодетект POI вдоль трека при загрузке** с адаптивным радиусом:
   - 50 м для точечных POI;
   - 100 м для линейных;
   - `ST_Intersects` для полигональных.
   Триггер: `ST_DWithin(track.geometry, location.geometry, radius)` или `ST_Intersects`.
5. **Заявка от загрузчика.** Загрузчик видит auto-detected POI read-only при загрузке. Может предложить новое POI модератору через заявку. Не создаёт сам — модерация обязательна.
6. **Low-zoom: server-side кластеризация POI** через global deterministic hex-grid;
   full per-POI — с zoom ~14 (см. A02).

## Альтернативы рассмотренные

- **Только точечные POI** (`geometry(Point)`). Отвергнуто: игнорирует «трек проходит через деревню» — деревня это административный полигон, не точка; не отвечает на запрос «через заповедник/хребет».
- **POI как теги внутри таксономии.** Отвергнуто: POI принципиально не тег (отношение much-to-many с порядком и расстоянием), теряет range-фильтр distance_along.
- **Только ссылки на OpenStreetMap/OSM, без собственного каталога.** Отвергнуто: external runtime-зависимость критического пути; удаление/перенос OSM-объекта падает на наш поиск; PostGIS-lookup всё равно нужises в кэше геометрии.
- **Free-form tag-cloud вместо каталога.** Отвергнуто — помойка через год.

## Последствия

- Положительные: self-contained каталог (не падает при OSM-изменениях); запрос
  «треки через локацию X» использует approved revision annotations; газеттир
  наполняется из OSM без runtime-зависимости; навигация «сверху вниз» решает low-zoom
  performance.
- Отрицательные: OSM fetch/cache/diff pipeline — дополнительная инфраструктура; модераторский UI для обновлений из OSM; fetch от OSM (Overpass/planet extracts) — external dependency при наполнении, не при runtime.
