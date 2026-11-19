<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 EON contributors
-->

# Offline-first sync — v0

Every device keeps a full copy of the person's documents and works without a network.
Devices exchange **changes**; how the changes travel is not part of this contract.

## Clock stamps

Every write carries a hybrid logical clock stamp (Kulkarni et al., 2014):

```
0001763539200000-0000-3f2a9c41d07e88b1
└ wall ms, 16 digits ┘ └ctr┘ └ node id ┘
```

- Stamps compare correctly **as strings**.
- A device's stamps never go backwards, even if its wall clock does.
- Importing a stamp moves the importing device's clock past it, so the next local edit is
  ordered after everything it has seen.
- The node id (random, per installation) breaks ties: two devices never produce the same
  stamp.
- A stamp more than **24 hours** ahead of the importing device's wall clock is refused, so
  one device with a wildly wrong clock cannot win every future merge.

## Changes

```json
{ "app": "org.eon.office.notes", "collection": "notes", "id": "n1",
  "doc": { "title": "Shopping" }, "deleted": false, "hlc": "0001763539200000-0000-3f2a9c41d07e88b1" }
```

A deletion is `"deleted": true, "doc": null`. `sync.export` returns changes after `since`,
oldest first, and `until` — the stamp to pass as `since` next time. `complete: false` means
there are more.

## Merging

Per document:

1. Same stamp already present → nothing to do (`unchanged`). Importing twice is harmless.
2. Incoming stamp **later** → it replaces the local version. If the local version was not
   already deleted, the local version is recorded as a **conflict**.
3. Incoming stamp **earlier** → the local version stays; the incoming version is recorded
   as a conflict.

Last-writer-wins at document granularity gives every device the same answer. Recording the
loser means no edit disappears silently: the application reads `storage.conflicts`, shows
or merges the losing version in a way that makes sense for *its* data (a paragraph is not a
spreadsheet cell), and calls `storage.resolve`. Field-level merging is deliberately left
to applications.

## Transports

- **v0:** any. Export to a file, carry it, import it. The core's tests do exactly this
  between two cores.
- **Planned:** `eon-office-server` as a relay an organisation or a person runs. End-to-end
  encryption of changes in transit through a server is a design requirement for it, not
  an afterthought; it is not specified yet.
