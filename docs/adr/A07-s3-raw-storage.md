# ADR A07 — S3-compatible storage for raw GPX and photos

**Status:** Accepted
**Date:** 2026-07-25

## Context

PostGIS stores processed geometry and metadata, but TrailBase also needs durable
original uploads, sanitized exports and photo objects. Blob storage does not belong in
the relational database.

## Decision

- Store exact private raw GPX bytes and photos in S3-compatible object storage.
- Store parsed geometry, metadata, ownership and lifecycle state in PostgreSQL/PostGIS.
- Use opaque object keys, private SHA-256 integrity checks and asynchronous upload/parse
  orchestration.
- Do not expose raw uploads through public or owner download routes; publish a separate
  sanitized export where appropriate.
- Use `aws-simple-sign` for SigV4 signing and `hato` for transport, keeping one HTTP
  stack for external integrations.

The normative rules are in:

- [GPX intake, geometry and object storage](../contract/tracks/ingest-storage.md)
- [Tracks, revisions and public exports](../contract/tracks/revisions-exports.md)

## Alternatives considered

- Database `bytea` — rejected because blob replication and backup costs do not fit the
  expected workload.
- AWS SDK v2 or SDK-v1 wrappers — rejected as unnecessary for the narrow operation set.
- Filesystem mounts — acceptable only for a throwaway prototype and not the chosen
  multi-node-ready direction.

## Consequences

- Upload flows must coordinate object storage and PostgreSQL lifecycle state.
- Operators need S3-compatible storage and its retention controls.
- The project keeps raw bytes private while exposing separately generated public data.
