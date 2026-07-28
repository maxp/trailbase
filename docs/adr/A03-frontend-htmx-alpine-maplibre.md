# ADR A03 — Фронтенд: htmx + alpine.js + MapLibre GL JS как остров

**Status**: Accepted
**Date**: 2026-07-25

**Уточнение 2026-07-28:** POI clustering выполняется только server-side; MapLibre не
кластеризует уже агрегированные features повторно. Полный frontend/map contract:
[Implementation Contract](../IMPLEMENTATION-CONTRACT.md#17-map-delivery-и-browser-state).

## Контекст

Слои фронтенда TrailBase: (а) auth-зона (server-side session cookie — см. A01); (б) поиск по фасетам + instant-search (см. A05); (в) каталог-списки и детали треков; (г) карта с basemap + адаптивными треками и кластерами POI (см. A02); (д) формы загрузки треков; (е) профиль высоты / галерея фото; (ж) модерация-инбокс.

Требуется стек фронтенда, который: сочетается с server-side cookies (A01), избегает SPA-накладных, рендерит 10k+ POI clusters + 10k+ упрощённых полилий в браузере (A02), остаётся сопровождаемым для open-source контрибьюторов. Пользователь прямо запросил htmx (после сравнения со SPA) и отдельно — исследование map-библиотек (Leaflet/OpenLayers/MapLibre/Deck.gl/Protomaps).

## Решение

1. **htmx** — основной слой UI. Server-rendered partials, заменяемые по `hx-get`/`hx-post`. Покрывает поиск/фасеты/списки/детали/CRUD-формы/навигацию.
2. **alpine.js** — дополнение для transient client state и моста к map-острову:
   - `Alpine.store('map')` — shared state между htmx-зоной и MapLibre instance; `map.on('moveend')` → обновление store → `htmx.ajax` для sidebar + `map.getSource().setData()` для треков.
   - Фильтр-группы (`x-show`), активные вкладки профиля трека, init form с in-flight state, lightbox-галерея, inline-формы модерации.
3. **MapLibre GL JS** — JS-остров. Персистентный DOM-узел вне htmx-swap-области; WebGL-рендер.
   - **POI clustering**: PostGIS возвращает готовые hex-grid cluster features; GeoJSON source использует `cluster: false`.
   - **Track polylines**: WebGL-рендер адаптивных GeoJSON, data-driven стиль (цвет по activity/difficulty).
4. **Bridge glue** (~100 строк): события карты → alpine store → htmx/ajax; htmx-события (hover трек в списке) → `map.setFeatureState`. Контейнер карты помечен `hx-disable` (htmx не свапает вглубь).

## Альтернативы рассмотренные (фронтенд-фреймворк)

- **SPA (React/Vue/Svelte) + MapLibre.** Отвергнуто: SPA была избыточной прослойкой поверх server-side cookie auth; добавляет build pipeline, client state management, дублирование схемы. Приёмлемо при future-переходе к tight-coupled map↔UI interactions (collaborative editing, drag-drop track editing), но не для scope 2.
- **htmx без alpine (zero-JS).** Отвергнуто: map-мост требует reactive shared state между картой и htmx-зоной без глобальных переменных; transient UI (фильтр-сворачивание, tabs, lightbox) через чистый htmx = лишние server round-trip.

## Альтернативы рассмотренные (map-библиотека)

- **Leaflet.** DOM/Canvas; markercluster держит 10k–50k POI, но лаг沿线 на 2000–3000 LineString при pan/zoom. Подходит для POI, но спотыкается о ключевую задачу — плотный слой треков.
- **OpenLayers.** Canvas/WebGL; возможно лучший по perf для LineString (MDPI 2025). Отвергнут из-за verbose API и steep learning curve для SPA-bridge — избыточен при htmx-доминанте.
- **Deck.gl.** WebGL; overkill (миллионы точек, 3D), требует подложки MapLibre/Leaflet-bottom.
- **Protomaps.** Требует PMTiles-формат basemap — противоречит требованию готовых OSM-тайлов.

## Последствия

- Положительные: маленький bundle, быстрая first-paint, server-rendered partials совпадают с cookie-auth (A01); один map-API для POI clusters и polylines; WebGL cap убирает DOM-краш.
- Отрицательные: два пути данных (HTML partial + raw GeoJSON) — каждый scoped interaction (hover в списке → highlight на карте, fly-to из search) — явный bridge-код; при росте числа interactions линейно растёт cost glue; при каком-то пороге окупается переход к SPA (без слома backend — REST API готов).
