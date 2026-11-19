<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 EON contributors
-->

# EON Office — ecosystem architecture

**Independent applications. One shared library. Separate repositories. One interoperable
ecosystem.** This page says what each of those words means in practice, and which contract
holds each boundary. The decision behind it is madde 43 of the project plan; the choice of
this shape over a core service is `eon-docs/office/design/2026-11-19-library-core.md`.

## The pieces

| Repository | Role | Licence | EON Stream counterpart |
|---|---|---|---|
| `eon-office-spec` (this) | Contracts: shared folder, manifest, capabilities, links, sync, themes, marketplace | Apache-2.0 | `eon-stream-spec` |
| `eon-office-core` | The library every application links | GPL-3.0-or-later + module exception | `eon-stream-core` |
| `eon-office-<app>` | One runnable application per repository (EON Notes, EON Tasks, …) | GPL-3.0-or-later + module exception | `eon-stream-app` |
| `eon-office-launcher` | The optional hub: installed applications, marketplace, permissions, preferences, recent items | GPL-3.0-or-later + module exception | — |
| `eon-office-sdk` | Tooling for application authors | MIT | `eon-stream-sdk` |
| `eon-office-marketplace` | Signed index of applications | CC0 index / Apache-2.0 tooling | `eon-stream-marketplace` |
| `eon-office-server` *(planned)* | Self-hostable sync relay and organisation directory | AGPL-3.0-or-later | `eon-edu-server` |

## Layers

```
 ┌───── applications: separate programs, separate repositories, separate installs ─────┐
 │  EON Notes      EON Tasks      EON Contacts      EON Calendar      EON Launcher (hub)  │
 │  └ core lib     └ core lib     └ core lib        └ core lib        └ core lib          │
 └────────┬──────────────┬──────────────┬────────────────┬──────────────────┬───────────┘
          │              └──────── <home>/shared.sqlite (shared-store.md) ───┘
   <home>/apps/<id>/data.sqlite — each application's own documents
                     │  sync format (sync.md)
 ┌───────────────────┴──────────────────────────────────────────────────────────────────┐
 │ eon-office-server (planned; self-hosted by an organisation, or by a person)            │
 └────────────────────────────────────────────────────────────────────────────────────────┘
```

## Rules that hold the boundaries

1. **Every application stands alone.** Installing EON Notes is all it takes to use EON
   Notes. Nothing has to be running first; no other application has to be installed.
2. **An application depends on the contracts, never on another application's code.** It
   reaches another application through an [`eon-office://` link](shared-store.md#5-links)
   or through the summaries that application [publishes](shared-store.md#3-sharedsqlite--schema-1).
   Both are declared in manifests (`provides`, `publishes`), so every cross-application
   interface is documented by construction.
3. **Every integration is optional.** A link to an application that is not installed is
   answered `not_installed`; published data that is not there is an empty list.
4. **The library holds no application concepts.** It stores documents; it does not know
   they are notes. A mechanism two applications need and neither should own becomes a
   library feature — through a pull request to this repository first.
5. **Data belongs to the application that wrote it, and to the person.** Each
   application's documents are its own file. The person can back up, restore and sync
   everything.
6. **Failure stays where it happened.** Separate processes, separate files; a newer shared
   file disables shared features, never an application's own documents.
7. **Notifications and reminders are the operating system's and the application's.**
   No EON component runs in the background to deliver them.

## Deployment modes

| Mode | What runs | Status |
|---|---|---|
| **Local** | Applications on one device. No network, no account. | Available (`v0`) |
| **Local + sync between own devices** | Export/import through a file or any transport ([sync.md](sync.md)) | Mechanism available; no automatic transport yet |
| **Self-hosted** | `eon-office-server` run by an organisation or a person | Planned |
| **EON-operated cloud** | The project runs the server for others | **Not before madde 30's Faz L1 gate** |

## What this architecture will not do

- Require a background service for anything to work.
- Load third-party code into an application (not before madde 4's sandbox design exists).
- Collect telemetry (madde 36).
- Bundle a mandatory AI provider (madde 43e).
- Combine applications into one program with tabs.
