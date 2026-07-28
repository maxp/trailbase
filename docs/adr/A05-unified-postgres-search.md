# ADR A05 — Единый поиск в PostgreSQL (текст × гео × фасеты × POI-join)

**Status**: Accepted
**Date**: 2026-07-25

**Уточнение 2026-07-28:** HTML и JSON имеют явные endpoints, pagination keyset,
все facet counts считаются server-side с disjunctive semantics. Полный контракт:
[Implementation Contract](../IMPLEMENTATION-CONTRACT.md#18-search-contract).

## Контекст

В каталоге TrailBase сосуществуют четыре гомогенно разных типа запросов: текстовый (`"вершина федосеева"`), гео (bbox / `ST_DWithin`), фасетный (`activity=hike AND difficulty=T3 AND season=winter`), реляционный («треки, проходящие через локацию X»). Каждый в Postgres имеет свой механизм (tsvector+GIN, GIST, compound btree, JOIN). Тип multiple backends даёт better search UX (instant fuzzy, ranking). Вопрос: один engine или split.

## Решение

1. **Единый search engine — PostgreSQL.** Без внешних поисковых систем на MVP.
2. **Механизмы:**
   - Текст: `tsvector` колонки, мультиязычная конфигурация (`russian`, `english`, `simple`) над jsonb-описаниями с языковым тегом; GIN-индекс; `pg_trgm` для fuzzy/опечаток.
   - Гео: GIST-индекс на `geometry`; `ST_Intersects`/`ST_DWithin`.
   - Фасеты: btree + compound indexes на `activity_type`, `difficulty`, `season`, `duration_source`.
   - POI-join: approved revision-location annotations; spatial predicates используются
     для autodetect, а не подменяют moderated catalog links.
3. **Склейка в PostgreSQL search service** через CTE; `/search` отдаёт HTML,
   `/api/v1/search` — JSON с keyset cursor.
4. **Instant-search** (`keyup changed delay:300ms`, <200ms target); все facet counts
   вычисляются server-side, исключая собственный facet filter.
5. **Путь наращивания:** при упоре в текстовый поиск закладывается переход на Meilisearch/Typesense (indexer postgress→Meilisearch через outbox/listen-notify) без слома schema.

## Альтернативы рассмотренные

- **PostGIS для гео/реляционного + внешний полнотекстовый index (Meilisearch/Typesense/OpenSearch).** Отвергнуто для MVP: второй datastore, pipeline синхронизации, eventual consistency. Приёмлемо позже при узком текстовом bottleneck.
- **pgvector для semantic search поверх tsvector.** Отвергнуто на MVP: умная фишка («покажи похожие треки»), но overreach vs работающий полнотекст.
- **Elasticsearch/OpenSearch полный стек.** Отвергнуто: enterprise-grade overhead (JVM, memory, ops cost) не оправдан для десятков тысяч треков.

## Последствия

- Положительные: self-contained (один datastore), без pipeline синхронизации; PostGIS + tsvector + GIN + GIST работает до ~100k треков при правильной индексации; мультиязычность (`russian`/`english`/`simple`) native.
- Отрицательные: fuzzy/опечаткоустойчивость `pg_trgm` посредственная vs Meilisearch; ranking без relevance-score-tuning; при росте текстового поиска — закладная на migration (но indexer из Postgres → Meilisearch накатывается без sloma schema).
