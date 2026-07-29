# ADR A08 — Backend на Clojure: стек и библиотеки

**Status**: Accepted
**Date**: 2026-07-25

**Уточнение 2026-07-28:** runtime — Java 25, Ring adapter — `http-kit`, handlers
синхронные, async work вынесен в Valkey-backed workers. Полный runtime contract:
[Implementation Contract](../IMPLEMENTATION-CONTRACT.md#1-runtime-и-границы-сервисов).

**Уточнение 2026-07-29:** Telegram/Max chats вызывают общие upload/search domain
services напрямую; deep-link token нужен только для optional browser session.
Короткоживущий chat-search callback state хранится в Valkey за случайным opaque ID.

## Контекст

TrailBase backend: HTTP-сервер, Telegram/Max chat adapters, messenger identity и
optional browser-session flow, парсинг GPX, S3-доступ, PostGIS-запросы (динамика
search + статика CRUD), server-rendered HTML partials для htmx, миграции. Chat и web
вызывают общие domain services. Выбор Clojure — данность (по пользователю). Требуется
зафиксировать идиоматический набор библиотек под уже принятые решения.

## Решение

| Слой | Библиотека | Обоснование |
|---|---|---|
| HTTP-сервер | **reitit** + ring + **http-kit** | Ring-compatible, компактный runtime; blocking/long work вынесен в отдельные workers |
| DB-доступ | **next.jdbc** | Современная JDBC-обёртка; reducible ResultSet; plain Clojure maps |
| SQL — динамика | **honeysql** | SQL-as-data; composable для фасетного search (A05), bbox+zoom queries (A02); PostGIS-функции как keys в maps |
| SQL — статика | **hugsql** | SQL-in-files для CRUD/модерации/миграций; читаемость, подсветка синтаксиса в `.sql` |
| Миграции | **migratus** | SQL-in-edn и `.sql` files, версионируются gitом; просто |
| Схемы/коерсия | **Malli** | data-driven, integrates с reitit coercion; валидация bot payloads/track upload/search params |
| HTML partials | **Hiccup** | hiccup-векторы — функции и есть partials; data-flow совместим с htmx (A03); без шаблонной системы |
| GPX parsing | **org.clojure/data.xml** | pull-parser, ленивый; 1-10 MB GPX — не bottleneck |
| HTTP-клиент | **hato** | единый стек для bot API, S3 (aws-simple-sign transport), external fetches; proxy support по требованию |
| S3-доступ | **aws-simple-sign** | SigV4 подпись, zero-dep; дополняет hato transport (см. A07) |
| JSON | **jsonista** | bot API payloads (Telegram/Max), client comms; быстрый Jackson-backed |
| Логирование | **telemere** | structured logs, modern Clojure-stack |
| Проект/tasks | **deps.edn** + babashka | современный tooling; bb tasks для миграций/dev ops |

## Альтернативы рассмотренные (внутри слоёв)

- **SQL: только honeysql.** Отвергнут: статические CRUD/модерация verbose как deep map-деревья; простые INSERT/SELECT громоздки.
- **SQL: только hugsql.** Отвергут: фасетный search (A05) динамический, не укладывается в статические сниппеты; либо duplication, либо fallback на honeql.
- **Amazonica.** Отвергнута: поверх AWS SDK v1 (EOL).
- **Cognitect aws-api.** Отвергнут: надстройка над SDK с медленным client creation; избыточен для узкого S3 API surface (см. A07).
- **Selmer (Jinja-style templates).** Отвергнут: разрывает data-flow с Clojure data; Hiccup-as-functions более idiomatic под htmx partials.
- **Compojure.** Отвергнут: устарел, не разрабатывается.
- **clojure.java.jdbc**. Отвергнут: deprecated, next.jdbc — приемник.

## Последствия

- Положительные: унифицированный HTTP (hato) для bot API + S3 + external; один прокси-механизм (по tight requirement); honeysql+hugsql — каждый в своей зоне; Malli schema = и для api-validation, и для domain-spec; Hiccup = чистый partial-data-flow.
- Отрицательные: Max bot library, почти наверняка не существует — raw Bot API через hato (Telegram тоже raw для единообразия); PostGIS функции — raw SQL в SQL-файлах hugsql, без Clojure-обёртки (правильно — GIS должен жить в PostGIS); GPX parsing больших файлов (>50MB) — pull-parser OK, но на краях возможно /// transplant в stream processor.
- Боты (Telegram/Max) — raw Bot API. Две обёртки = две точки drift; raw = единый stack bot→hato→jsonista→telemere.
