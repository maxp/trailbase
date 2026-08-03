# M03 — GPX Upload + Parse

## Goal

Accept a GPX upload from web or private chat, retain its original bytes privately,
parse it asynchronously and create a durable private track draft.

## Acceptance

- Valid GPX reaches a private draft through the complete upload and parse flow.
- Invalid input, quota limits and infrastructure failures preserve the stated lifecycle.
- Raw-object integrity and track-issue behavior are observable to the right actors.

## Contract

- [GPX intake, geometry и object storage](../contract/tracks/ingest-storage.md)
- [Tracks, revisions и public exports](../contract/tracks/revisions-exports.md)
