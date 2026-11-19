<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 EON contributors
-->

# EON Office — ecosystem architecture

**One shared foundation. Independent applications. Separate repositories. One
interoperable ecosystem.** This page says what each of those words means in practice, and
which contract holds each boundary. The decision behind it is madde 43 of the project plan.

## The pieces

| Repository | Role | Licence | Contract it implements or consumes |
|---|---|---|---|
| `eon-office-spec` (this) | Contracts: protocol, manifest, themes, capabilities | Apache-2.0 | — |
| `eon-office-core` | The local core service | GPL-3.0-or-later + module exception | implements the protocol |
| `eon-office-sdk` | Client library and templates for application authors | MIT | consumes the protocol |
| `eon-office-<app>` | One repository per application (EON Notes, EON Writer, …) | GPL-3.0-or-later | consumes the protocol; each declares its own `provides`/`publishes` |
| `eon-office-marketplace` *(planned)* | Signed index of applications; discovery, install, update | CC0 index / Apache-2.0 tooling | the manifest + signing (madde 34) |
| `eon-office-server` *(planned)* | Self-hostable server: sync relay, organisation directory, OIDC | AGPL-3.0-or-later | sync format, directory model |

Applications are named `EON <Name>` to people and `eon-office-<name>` as repositories
(madde 32a, 43). The roadmap that orders them is `eon-docs/office/roadmap.md`.

## Layers

```
 ┌──────────────── applications: separate processes, separate repos ─────────────────┐
 │  EON Notes   EON Writer   EON Sheets   EON Tasks   EON Mail   …   third parties     │
 │      └────────────── eon-office-sdk (optional; the protocol is the contract) ──┘    │
 └───────────────────────────────────────┬────────────────────────────────────────────┘
                     JSON-RPC over loopback (protocol.md) │
 ┌───────────────────────────────────────┴────────────────────────────────────────────┐
 │ eon-office-core (one per person per device)                                         │
 │  registry · capabilities · directory · storage · sync · events · calls · search ·   │
 │  notifications · jobs · settings/prefs/themes · audit · backup                      │
 └───────────────────────────────────────┬────────────────────────────────────────────┘
               sync format (sync.md)      │  (planned) directory, OIDC
 ┌───────────────────────────────────────┴────────────────────────────────────────────┐
 │ eon-office-server (planned; self-hosted by an organisation, or by a person)         │
 └─────────────────────────────────────────────────────────────────────────────────────┘
```

## Rules that hold the boundaries

1. **An application depends on the protocol, never on another application's code.** It
   reaches another application only through methods that application declares in
   `provides`, via `apps.call`, and through events it declares in `publishes`. Both lists
   are in the manifest, so every cross-application interface is documented by
   construction.
2. **Every integration is optional.** `apps.available` says what is installed and running;
   `apps.call` answers `app_unavailable` when the other side is absent. An application
   must stay fully usable without any other application installed (development rule 15,
   16). EON Notes without EON Tasks is EON Notes.
3. **The core holds no application concepts.** It stores documents; it does not know they
   are notes. When two applications need the same mechanism and neither should own it, it
   becomes a core feature — through a pull request to this repository first.
4. **Data belongs to the application that wrote it, and to the person.** Applications
   cannot read each other's storage. The person (owner) can export, back up, restore and
   sync everything. Uninstalling an application deletes what the core held for it.
5. **Failure stays where it happened.** Separate processes; bounded queues; timeouts on
   calls; supervision with a restart limit.
6. **Nothing is granted by installing.** Capabilities are requested in the manifest and
   granted by the owner. AI features follow the same rule: an assistant application gets
   data only through calls the person allowed (madde 43e).

## Deployment modes

| Mode | What runs | Status |
|---|---|---|
| **Local** | The core and applications on one device. No network, no account. | Available (`v0`) |
| **Local + sync between own devices** | Two cores exchanging `sync.export`/`sync.import` through a file or any transport | Available as a mechanism; no automatic transport yet |
| **Self-hosted** | `eon-office-server` run by an organisation or a person; cores sync through it; the organisation's directory and sign-in | Planned |
| **EON-operated cloud** | The project runs the server for others | **Not before madde 30's Faz L1 gate** — a legal entity, legal review, and the rest of its five conditions |

The person chooses where their data lives; the default is their own device.

## What this architecture will not do

- Load application code into the core (no plugins in the core until madde 4's sandbox
  design exists).
- Collect telemetry (madde 36).
- Bundle a mandatory AI provider (madde 43e).
- Combine applications into one binary with tabs. The independent repository and the
  independent install are the product.
