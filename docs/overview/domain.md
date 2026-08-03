# TrailBase — Domain map

## Core concepts

- **Account** owns identities, tracks and preferences.
- **Identity** is a Telegram or Max user binding; it permits private-chat operations.
- **Browser session** is optional and never replaces the messenger identity model.
- **Track** is the stable aggregate; its published and draft content lives in immutable
  **revisions**.
- **Raw object** is the exact private uploaded source; a sanitized export is a distinct
  derived object.
- **POI** is a curated geographical entity that may relate to tracks.
- **Track issue** is an admin-only integrity finding that derives capability blocks from
  its code.

## Main flows

1. A private web or chat upload creates a raw object and an asynchronous parse flow.
2. Parsing produces a private revision; moderation may make it public.
3. Public catalog, map and search expose only data allowed by the publication and
   integrity boundaries.
4. Bot and web adapters call the same domain services and permission checks.

Detailed behavior belongs to the matching [contract topic](../IMPLEMENTATION-CONTRACT.md).
