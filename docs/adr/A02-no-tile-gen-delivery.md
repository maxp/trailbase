# ADR A02 — Доставка геоданных: готовые OSM-тайлы + adaptive GeoJSON, без генерации тайлов

**Status**: Accepted
**Date**: 2026-07-25

**Уточнение 2026-07-28:** canonical geometry — 2D MultiLineString в immutable track
revision; точная zoom-матрица и tolerances зафиксированы в
[Implementation Contract](../IMPLEMENTATION-CONTRACT.md#17-map-delivery-и-browser-state).

**Уточнение 2026-07-30:** один GPX attachment создаёт один draft, включая файл с
несколькими `<trk>`. Каждый валидный `<trkseg>` остаётся отдельным компонентом
canonical MultiLineString в document order. Отдельный segment manifest не хранится;
исходная hierarchy остаётся в original raw GPX. Multi-track/route parse сохраняет
только scalar track/route/valid-segment counts и показывает неблокирующий warning
рядом с preview; отдельного confirmation gate до final submit нет.

## Контекст

TrailBase показывает треки на карте. Требуется архитектура доставки геоданных, которая: (а) использует **готовые OpenStreetMap-тайлы** (по прямому требованию пользователя — «не генерируем тайлы»); (б) масштабируется на десятки тысяч треков; (в) не требует генерации MVT/раstup-tile pipeline (Martin, pg_tileserv, tileserver-gl отKeyep на ROI).

Без tile-gen треки должны рендериться **на клиенте как векторный оверлей** поверх OSM-растра — критическое constraint при ~10k+ треков.

## Решение

1. **Basemap = готовые OSM raster-тайлы** как `raster`-source в MapLibre. Без собственной генерации, без tileserver, без mbtiles pipeline.
2. **Треки доставляются как adaptive GeoJSON по bbox + zoom:**
   - **Low-zoom (≤12)**: _треки не рисуются_; рисуются только **POI кластеры** из deterministic server-side hex-grid.
   - **Mid-zoom (13–15)**: z13→z11, z14→z13, z15→z15 precomputed geometry.
   - **High-zoom (≥16)**: z15 context layer плюс full geometry выбранного трека.
3. **Pre-computed упрощённые геометрии** в PostGIS: 3 уровня (z11/z13/z15) в колонках `geometry_simplified_z11/13/15` при загрузке трека. On-the-fly `ST_Simplify` только как fallback.
4. **Запрос:** `/api/tracks?bbox=...&zoom=...` → PostGIS `ST_Intersects(geometry, bbox)` + zoom-aware выбор колонки.
5. **Raw GPX-файлы + фото — в S3** (см. A07). PostGIS хранит только распарсенную geometry и метаданные.
6. **Сегменты GPX**: один attachment не разделяется автоматически на несколько
   drafts. Компоненты `MultiLineString` следуют document order и сами сохраняют
   segment boundaries. Исходная `<trk>/<trkseg>` hierarchy отдельно не
   материализуется и при необходимости читается повторно из original raw GPX.
   Route-only GPX использует те же one-attachment/one-draft правила. Scalar
   source-track/source-route/valid-segment counts питают неблокирующий draft/final
   warning; final submit является единственным confirmation всего snapshot.

## Альтернативы рассмотренные

- **PostGIS + Martin (MVT on-the-fly).** Отвергнуто пользователем: «не генерируем тайлы».
- **pg_tileserv.** То же ограничение + более passive community.
- **Server-rendered raster overlay-tiles (WMS).** Технически «тайлы» в широком смысле — противоречит требованию.
- **Полная GeoJSON-выгрузка каталога.** Отвергнуто: crash брауser at >2k треков.
- **Bbox-фильтрованный запрос без zoom-aware simplification.** Недостаточно на низких zoomах (bbox wide → тысячи треков).

## Последствия

- Положительные: ноль tile-gen инфраструктуры (нет Martin/pg_tileserv/tileserver-gl/mbtiles); максимум ~10k упрощённых полилиний рендерится в MapLibre WebGL без лагов; PostGIS GIST-индекс делает bbox-lookup cheap.
- Отрицательные: клиентский рендер состоит из **двух путей данных** (HTML partials через htmx для sidebar + raw GeoJSON через fetch для карты); bridge glue код ~100 строк (см. A03); pre-compute колонок adds storage cost per трек.
- Низкие zoom'ы агрегируются через POI-кластер (см. A04) — навигация каталога «сверху вниз»: газеттир → треки через локацию.
