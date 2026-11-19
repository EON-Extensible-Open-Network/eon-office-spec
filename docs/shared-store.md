<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 EON contributors
-->

# The shared folder — v0 (UNSTABLE)

How independently installed EON Office applications find and help each other without any
of them — or anything else — having to run. This replaces the core service protocol
([protocol.md](protocol.md), kept for history). The decision and what it costs:
`eon-docs/office/design/2026-11-19-library-core.md` (madde 43).

The reference implementation is
[`eon-office-core`](https://github.com/EON-Extensible-Open-Network/eon-office-core), a
library; its `tests/library.rs` exercises every section below.

## 1. The home folder

`EON_OFFICE_HOME` if set and non-empty; otherwise

| Platform | Folder |
|---|---|
| Windows | `%LOCALAPPDATA%\EON\Office` |
| macOS | `~/Library/Application Support/EON Office` |
| Others | `$XDG_DATA_HOME/eon-office`, else `~/.local/share/eon-office` |

```
<home>/shared.sqlite          what applications share (§3)
<home>/apps/<id>/data.sqlite  one application's documents (§4); only it opens this file
<home>/apps/<id>/             the application's other files
```

## 2. Opening the files

SQLite 3, `journal_mode=WAL`, `synchronous=NORMAL`, `foreign_keys=ON`, a busy timeout of
**5 seconds**. Several processes open `shared.sqlite` at once, so:

- every transaction that reads and then writes **begins `IMMEDIATE`** (a deferred one that
  has read cannot wait for the write lock and fails when another process wrote);
- every write is short; nothing holds a transaction across user interaction;
- the schema version is `PRAGMA user_version`. A reader that finds a **higher** version than
  it knows does not touch the file. For `shared.sqlite` that disables the shared features
  only; the application keeps working with its own documents.

## 3. `shared.sqlite` — schema 1

| Table | Columns | Rules |
|---|---|---|
| `registry` | `id` PK, `manifest` (JSON), `path` (folder of the program, nullable), `revoked` (JSON array of capability strings, default `[]`), `registered_at`, `updated_at` (ms) | An application upserts its own row at start. A lower `version` than the stored one does not replace the row. `revoked` is changed by the person only. |
| `settings` | `scope`, `key`, `value` (JSON); PK `(scope, key)` | Scope `prefs`: the person's preferences — `locale` (BCP 47 or `system`), `theme` (`system`, `light`, `dark` or a theme id), `reduce_motion`, `high_contrast` (booleans), `font_scale` (0.75–2.0). |
| `meta` | `key` PK, `value` | `prefs_version`: an integer incremented in the same transaction as every preference change. Applications poll it (at most every 2 s) to notice a change. |
| `themes` | `id` PK, `body` (a [theme](../schemas/theme.v0.schema.json)) | Refused unless every text/background pair meets WCAG AA. |
| `search_docs` | `app`, `id` (PK pair), `title`, `snippet` (first 160 characters), `visibility` (`private`/`shared`), `workspace`, `updated_at` | An application writes only its own rows. An application sees its own rows and every `shared` row; the person's hub sees all. |
| `search_terms` | `term`, `app`, `id`, `weight`; PK `(term, app, id)` | Terms are Turkish-folded ([§6](#6-text-folding)); title terms weigh 3, body terms 1. Replaced with the doc row in one transaction. |
| `activity` | `seq` PK, `at`, `app`, `item`, `title`, `kind` | Ids and titles, **never contents**. Rows older than the newest 500 are deleted in the inserting transaction. |
| `public` | `app`, `key` (PK pair), `value` (JSON, ≤ 16 KiB), `updated_at` | Summaries an application offers others (Contacts: name, organisation, addresses, phones). Written only by `app`; read by applications that declare `call:<app>`. |
| `audit` | `seq` PK, `at`, `actor`, `action`, `target`, `detail`, `prev`, `hash` | `hash = sha256(prev \n seq \n at \n actor \n action \n target \n detail)`, hex; the first `prev` is 64 zeros. Actions: `app.registered`, `app.upgraded`, `app.unregistered`, `capability.revoked`, `capability.granted`. |
| `users`, `orgs`, `workspaces`, `teams`, `memberships` | as in `eon-office-core/src/db.rs` | The directory, used once `eon-office-server` brings organisations. |

## 4. `apps/<id>/data.sqlite` — schema 1

| Table | Columns | Rules |
|---|---|---|
| `meta` | `key` PK, `value` | `node`: 16 hex characters, this device's identity in [sync](sync.md). |
| `docs` | `app`, `collection`, `id` (PK), `body` (JSON or null), `rev`, `hlc`, `deleted` | `app` is always the folder's application id. Writes carry an expected `rev` (`0` = must not exist); a mismatch is a `conflict`. Deletes leave a tombstone. Collections and ids match `^[A-Za-z0-9._:-]{1,128}$`; a body is at most 512 KiB. |
| `conflicts` | `id` PK, `app`, `collection`, `doc_id`, `body`, `deleted`, `hlc`, `winner_hlc`, `at` | The losing side of every sync merge, until the application resolves it. |
| `settings` | `scope`, `key`, `value` | The application's own settings, scope = its id. |

## 5. Links

```
eon-office://<app-id>/<item>[?<query>]
```

`<app-id>` is a valid application id; `<item>` has no whitespace or control characters and
is at most 512 bytes. To deliver a link, start the program the registry names —
`path` joined with `manifest.entry.command`, trying the name as written and then with the
platform's executable suffix — with the link as its **only argument** and
`EON_OFFICE_HOME` set. An application must be single-instance: a second start hands its
argument to the running window and exits.

An application treats `<item>` as the id of something to show. It never runs it, never
fetches it, and does not act on a query parameter that would change data without the
person confirming.

## 6. Text folding

Search terms are folded so a person finds what they meant on any keyboard (madde 40):
every i (`I ı İ i` and accented forms) becomes `i`; `ş ğ ç ö ü` become `s g c o u`; common
Latin accents are dropped; combining marks (U+0300–U+036F) vanish; everything else is
lower-cased. Terms are runs of letters and digits, at most 48 characters. A query matches
a term it is a prefix of; every query term must match. The reference is
`eon-office-core/src/text.rs`. Sorting is the application's: folding is lossy and would
sort `ı` with `i`.

## 7. Capabilities

What each capability allows is in [capabilities.md](capabilities.md). A library or
application checks the manifest's declaration minus the registry's `revoked` before each
use. That is transparency and a guard against mistakes, not a boundary against a program
running as the same person.
