<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 EON contributors
-->

# Versioning and compatibility

EON Office is many independently versioned components. These are the numbers that decide
whether two of them work together.

| What | Form | Today | Where it is checked |
|---|---|---|---|
| **Protocol** (`api`) | integer | `0` | `session.hello` (`api`), manifest `api.min..=api.max`, `endpoint.json` |
| **Manifest format** (`manifest`) | integer | `0` | `apps.install` |
| **Theme format** | per schema file | `v0` | `themes.put` |
| **Core database schema** | integer (`PRAGMA user_version`) | `1` | core start-up, `restore` |
| **Each repository's release** | semver | `0.x` | its own changelog |
| **Each application's version** | semver, in its manifest | — | `apps.install` refuses a downgrade |

## Rules

- **`v0` is unstable.** Any of the above may break between releases until `v1`, without a
  migration path (plan, "Kapsam notu"). Breaking changes are listed in `CHANGELOG.md`.
- **From `v1` on:** additive changes (a new method, a new optional parameter, a new event,
  a new result field) keep the protocol number. Removing or changing the meaning of
  anything raises it. A core supports at least the current and the previous protocol
  version for one year after a raise.
- **Applications must ignore fields they do not know** in results and events. The core
  does the same for nothing: it refuses unknown manifest fields, because a typo in a
  permission list should be an error.
- **Independent releases.** A new core does not require new applications, and a new
  application does not require a new core, as long as the `api` ranges overlap. The
  marketplace (planned) checks this before offering an update.
- **No downgrades of installed applications** (madde 39). Rolling back is uninstall and
  reinstall, which the person sees.
- **A database written by a newer core is refused** by an older one rather than opened.
