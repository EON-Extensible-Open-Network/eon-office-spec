<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 EON contributors
-->

# eon-office-spec

**The contracts of EON Office.** How an application talks to the core, what its manifest
says, what a capability means, how devices sync, how versions relate. Everything other
people's work depends on lives here, and is licensed permissively so that anyone — including
clients and applications this project will never control — can implement it.

> **Contracts `v0` — UNSTABLE.** They may break without a migration path until `v1`. The
> decision to build EON Office is madde 43 of the
> [project plan](https://github.com/EON-Extensible-Open-Network/eon-docs/blob/main/plan/eon-plan.md).

| Document | What it fixes |
|---|---|
| [docs/architecture.md](docs/architecture.md) | The repositories, the layers, the rules that keep applications independent, deployment modes |
| [docs/protocol.md](docs/protocol.md) | Transport, sessions, every method, every error, every event |
| [docs/capabilities.md](docs/capabilities.md) | What each permission allows and how to explain it to a person |
| [docs/sync.md](docs/sync.md) | Clock stamps, change format, merge rules, conflicts |
| [docs/versioning.md](docs/versioning.md) | Which numbers decide compatibility |
| [schemas/app-manifest.v0.schema.json](schemas/app-manifest.v0.schema.json) | The application manifest |
| [schemas/message.v0.schema.json](schemas/message.v0.schema.json) | The message envelope on the wire |
| [schemas/theme.v0.schema.json](schemas/theme.v0.schema.json) | Declarative themes |

The reference implementation is
[`eon-office-core`](https://github.com/EON-Extensible-Open-Network/eon-office-core); the
client library is [`eon-office-sdk`](https://github.com/EON-Extensible-Open-Network/eon-office-sdk).

## Validate

```
npm ci
npm run validate
```

Every file in `examples/valid` must pass its schema and every file in `examples/invalid`
must fail — a schema that accepts everything would otherwise pass. `eon-office-core` runs
the same examples through its own readers in CI, so the schema and the implementation
cannot drift apart silently.

Schema `$id`s assume the domain `spec.eonstream.org`; madde 32g (name and domain
availability) is still open and may change them.

## Changing a contract

Open a pull request here first; implementations follow. Say what changes, who it breaks,
and why the change is worth that. A new capability also states the one line a person will
be shown when asked to grant it ([capabilities.md](docs/capabilities.md)).

Licence: Apache-2.0. Contributing: [CONTRIBUTING.md](CONTRIBUTING.md). Security:
[SECURITY.md](SECURITY.md).
