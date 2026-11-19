<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 EON contributors
-->

# Changelog

## Unreleased

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
