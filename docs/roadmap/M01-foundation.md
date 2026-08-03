# M01 — Foundation

## Goal

Deploy the minimal runnable platform: Clojure services, PostgreSQL/PostGIS, Valkey,
object storage, migrations, configuration and observable health endpoints.

## Acceptance

- A fresh environment starts through the documented deployment path.
- Migrations and service health are observable.
- The base security, configuration and logging invariants are met.

## Contract

- [Runtime и platform](../contract/runtime.md)
- [Health, logs и metrics](../contract/observability.md)
- [Frontend build, migrations и deploy](../contract/platform.md)
