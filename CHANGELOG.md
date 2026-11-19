<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 EON contributors
-->

# Changelog

## Unreleased

- **The shared folder replaces the core service** ([shared-store.md](docs/shared-store.md)):
  `shared.sqlite` (registry, preferences, themes, search, activity, published summaries,
  audit) and `apps/<id>/data.sqlite`, opened by every application through the
  `eon-office-core` library. `protocol.md` and the message schema are kept, marked
  superseded. `architecture.md` and `capabilities.md` rewritten: capabilities are declared,
  shown at install, revocable, and described as transparency rather than a sandbox;
  `events.publish` covers the recent list and published summaries, `call:<id>` covers
  links and reading `<id>`'s summaries. Links are delivered by starting the registered
  program with the link as its only argument.

- Protocol: the well-known `activity` event (recent items and the activity feed, titles
  never contents) and the `eon-office://` deep-link format.
- Marketplace: entry schema v0 and `docs/marketplace.md` (index checks, client install,
  update and signature rules; ratings and verification stated as gated, not invented).
- Protocol: well-known methods `open` and `focus`, so a shell can open a search result
  or bring an application forward without knowing it.

## 0.1.0 — contracts v0

- Protocol `v0`: loopback TCP, JSON-RPC 2.0 one object per line, sessions for the owner
  and for applications, authority checked per call, fifteen stable error codes, the method
  catalogue for applications, directory, storage, sync, events, calls, notifications,
  search, jobs, settings, preferences, themes, audit and backup.
- Application manifest format `0`, with capabilities, `provides`, `publishes`, `entry`,
  `restart`.
- Message envelope and theme schemas.
- Sync: hybrid logical clock stamps, change format, last-writer-wins with losers kept.
- Architecture, capabilities and versioning documents.
- 8 valid and 17 invalid examples; `npm run validate`.
