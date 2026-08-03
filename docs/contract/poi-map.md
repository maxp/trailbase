# Контракт: POI и map delivery

Часть [Implementation Contract](../IMPLEMENTATION-CONTRACT.md). При противоречии с другими документами действует этот контракт.

## 16. POI/gazetteer

- `locations` хранит stable identity и `current_revision_id`.
- Geometry, names, category и OSM snapshot находятся в immutable
  `location_revisions`. OSM refresh создаёт pending revision и не перезаписывает
  ручные правки.
- Geometry kind (`point`, `line`, `area`) отделён от semantic `category_id`.
  Categories образуют модерируемый иерархический справочник.
- Основные названия хранятся в `name_i18n`; aliases — отдельно с language, source и
  normalized form. OSM names/alt names импортируются с provenance, moderator выбирает
  main display name.
- OSM import — async job с configurable Overpass endpoint, rate limit, retries и
  response cache. Moderator видит preview/diff и отдельно применяет revision.
- Provenance: `osm_type`, `osm_id`, `osm_version`, `osm_timestamp`, `fetched_at`,
  source URL и hash нормализованного ответа. `(osm_type, osm_id)` уникальна среди
  active locations.
- POI links — отдельно модерируемые annotations конкретной geometry revision, а не
  причина создавать новую track revision.
- Link хранит source (`autodetect`, `author_suggestion`, `moderator`), status
  (`suggested`, `approved`, `rejected`), confidence, reviewer и timestamps.
- Одна logical location link может иметь несколько occurrences. Каждая occurrence
  хранит `seq`, cumulative `distance_along_m`, `segment_index`,
  `segment_distance_m`. Разрыв между segments не добавляет расстояние.
- High-confidence связь с уже approved location публикуется автоматически:
  - point: расстояние до track не больше 25 м;
  - line: track находится в пределах 50 м не менее 100 м;
  - area: длина track внутри polygon не меньше 50 м.
- Остальные candidates в общих радиусах остаются `suggested`.
- Окончательно rejected autodetect link создаёт suppression для
  `(track_revision_id, location_id)`. Повторный запуск или новая версия алгоритма не
  добавляет связь снова; пересмотр выполняет moderator. Новая geometry revision трека
  оценивается независимо.
- Первый author dispute переводит auto-approved link в `disputed` и немедленно
  скрывает его до moderator decision. Если moderator проверил и восстановил link,
  следующий author flag создаёт обращение, но не скрывает verified link автоматически.
- Dispute требует reason code: `too_far`, `route_does_not_pass`, `wrong_location`,
  `duplicate`, `other`. Комментарий optional, кроме `other`; moderator decision всегда
  содержит причину.
- Новая location geometry revision запускает async revalidation всех связанных
  annotations:
  - autodetected links пересчитываются;
  - author/moderator links получают `needs_review`, если новая geometry им
    противоречит;
  - до результата публичной остаётся последняя approved связь со внутренним
    `revalidating`;
  - результат переключается атомарно.
- Revalidation использует пять retries с backoff, затем DLQ. Прежняя approved связь
  остаётся публичной с внутренним `revalidation_failed`; состояние дольше 24 часов
  вызывает operator/moderator alert.
- Duplicate locations объединяются moderator merge в выбранную canonical location:
  aliases/provenance переносятся, annotations перепривязываются с дедупликацией
  occurrences, старый UUID становится tombstone с permanent redirect, операция
  audit-ится.
- Location merge обратим 90 дней по immutable merge snapshot и mapping перенесённых
  annotations. После срока redirect постоянный, snapshot может быть удалён.
- Ошибочная связанная location переводится в `archived`: скрывается из карты, поиска
  и public links, URL остаётся tombstone, annotations идут на review. Несвязанная
  archived location физически очищается через 90 дней, сохраняя UUID tombstone,
  redirect/merge history и audit.
- Используемая semantic category не удаляется: она получает `deprecated` и
  `replacement_category_id`. Category key неизменяем.
- Category hierarchy — строгое дерево с одним `parent_id`. Родительский search filter
  включает descendants; JSON API поддерживает `category_exact`.
- Обычный пользователь предлагает seed point и optional OSM URL. Line/area geometry
  импортирует или создаёт moderator.
- Suggestion limits: 10 новых заявок в сутки и 20 одновременно pending на user,
  с admin override.
- Право `submit_location_suggestions` можно ограничить отдельно от аккаунта. Restriction
  audit-ится, содержит причину и optional expiry.
- Low-zoom clustering использует глобальную deterministic hex-grid в EPSG:3857.
  Client-side clustering отключён. Для line/area representative point —
  `ST_PointOnSurface`.

## 17. Map delivery и browser state

- Production tile provider выбирается отдельно; URL и attribution задаются
  конфигурацией. `tile.openstreetmap.org` допустим только для dev/test малой нагрузки.
- MapLibre attribution control всегда видим и содержит OSM contributors и выбранного
  provider. Auth page не загружает карту или tiles.
- Zoom matrix:
  - z0–12: только server-side POI clusters;
  - z13: `geometry_simplified_z11`;
  - z14: `geometry_simplified_z13`;
  - z15: `geometry_simplified_z15`;
  - z16+: z15 context layer плюс full geometry overlay выбранного track.
- Cluster feature содержит count и bbox. Click делает `fitBounds` с `maxZoom: 14`;
  одиночный POI открывает detail.
- `/api/tracks.geojson` возвращает максимум 1 000 features, `truncated: true` при
  переполнении и предлагает приблизить карту.
- Selection при переполнении: completeness, `published_at DESC`, UUID. Completeness —
  по одному баллу за description, difficulty, season, duration и approved POI; это
  не публичный quality rating.
- GeoJSON coordinates имеют шесть знаков после запятой. Properties минимальны:
  track ID, name, activity, difficulty code, length, published revision ID/time.
- Track под active full track lock не входит в GeoJSON, sidebar или другие catalog
  collection results.
- GeoJSON использует `ETag` и
  `Cache-Control: public, max-age=30, stale-while-revalidate=60`.
- Активация full track lock не purge-ит уже закэшированные GeoJSON responses: они
  могут содержать track до 90 секунд в пределах существующих cache directives;
  новые responses, увидевшие lock, track исключают.
- Viewport bbox расширяется наружу до фиксированной zoom-grid для повторного
  использования cache.
- Client делает debounce 150 мс, отменяет старый fetch через `AbortController` и
  отбрасывает stale responses по sequence number.
- URL state: `map=z/lat/lon`, отдельные filter params, канонический detail path
  `/tracks/:id`. Filter/map updates используют `replaceState`, явный detail —
  history entry.
- Каждая map entity имеет keyboard-accessible эквивалент в semantic sidebar.
- Без WebGL карта скрывается, но sidebar, search, detail и upload metadata остаются
  работоспособными.
- Public track detail полностью SSR: name, description, author, metrics, POI, license
  и download доступны без JavaScript.
- Прямой `/tracks/:id` для otherwise published track под active full track lock
  возвращает generic `503 Service Unavailable` с `Cache-Control: no-store`, без
  `Retry-After`, track metadata или integrity details. Private, archived и
  moderator-removed tracks сохраняют одинаковый `404`.

