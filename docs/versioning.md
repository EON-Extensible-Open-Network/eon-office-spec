<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 EON contributors
-->

# Versioning and compatibility

EON Office is many independently versioned components. These are the numbers that decide
whether two of them work together.

| What | Form | Today | Where it is checked |
|---|---|---|---|
| **Shared-store version** (`api`) | integer | `0` | manifest `api.min..=api.max`; the marketplace offers only releases whose range contains the installed library's |
| **Manifest format** (`manifest`) | integer | `0` | registration (`Home::app`), the marketplace |
| **Theme format** | per schema file | `v0` | `Shared::put_theme` |
| **`shared.sqlite` schema** | integer (`PRAGMA user_version`) | `1` | every application at start; a newer one disables shared features only |
| **`data.sqlite` schema** | integer (`PRAGMA user_version`) | `1` | the application at start, restore |
| **Each repository's release** | semver | `0.x` | its own changelog |
| **Each application's version** | semver, in its manifest | — | registration never replaces a newer registry row with an older one |

## Rules

- **`v0` is unstable.** Any of the above may break between releases until `v1`, without a
  migration path (plan, "Kapsam notu"). Breaking changes are listed in `CHANGELOG.md`.
- **From `v1` on:** additive changes (a new table, a new column with a default, a new
  optional manifest field) keep the shared-store number. Removing or changing the meaning
  of anything raises it, with a migration the library applies under the write lock.
- **Applications must ignore fields they do not know** in published summaries and
  activity. The library does the same for nothing: it refuses unknown manifest fields,
  because a typo in a permission list should be an error.
- **Independent releases.** A new core does not require new applications, and a new
  application does not require a new core, as long as the `api` ranges overlap. The
  marketplace (planned) checks this before offering an update.
- **No downgrades of installed applications** (madde 39). Rolling back is uninstall and
  reinstall, which the person sees.
- **A database written by a newer core is refused** by an older one rather than opened.
