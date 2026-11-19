<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 EON contributors
-->

# EON Office application protocol — v0 (UNSTABLE)

> **Superseded (19 November 2026).** EON Office no longer has a core service: the core is a
> library and every application stands alone. What applications share is specified in
> [shared-store.md](shared-store.md). This page is kept because `eon-office-core 0.1.0`,
> `eon-office-sdk 0.1.0` and the first application releases implement it; nothing new
> should.

How an application talks to the EON Office core. Everything an application can do with the
core is on this page; there is no other interface. `v0` may change without a migration
path until `v1` (see [versioning.md](versioning.md)).

The reference implementation is
[`eon-office-core`](https://github.com/EON-Extensible-Open-Network/eon-office-core); its
`tests/protocol.rs` exercises every section below with a bare socket.

## 1. Transport

- **TCP on the loopback interface.** The core never listens on any other address.
- **Framing:** one JSON value per line, UTF-8, terminated by `\n`. A line longer than
  **1 MiB** is answered with `limit` and the connection is closed.
- **Messages:** JSON-RPC 2.0 requests, notifications and responses
  ([`schemas/message.v0.schema.json`](../schemas/message.v0.schema.json)). Batches are not
  supported. `params` is an object.
- **Both directions.** The core also sends: notifications (`event`, `job.due`,
  `shutdown`) and requests (`invoke`), which the application answers with a normal response
  carrying the same `id`.

### Finding the core

The core writes `<home>/runtime/endpoint.json` while it runs:

```json
{ "api": 0, "address": "127.0.0.1:53187", "pid": 4120, "core_version": "0.1.0", "node": "3f2a9c41d07e88b1" }
```

`<home>` is `EON_OFFICE_HOME`, else `%LOCALAPPDATA%\EON\Office` (Windows),
`~/Library/Application Support/EON Office` (macOS), `$XDG_DATA_HOME/eon-office` or
`~/.local/share/eon-office` (elsewhere). An application the core launched also gets
`EON_OFFICE_ADDRESS`, `EON_OFFICE_APP`, `EON_OFFICE_TOKEN` and `EON_OFFICE_HOME` in its
environment.

## 2. Sessions and principals

The first request on a connection must be `session.hello` (only `core.info` is answered
before it). There are two kinds of principal:

| Principal | Token | Who |
|---|---|---|
| **owner** | `<home>/runtime/owner.token` | The person, through their shell or administration tool |
| **app** | `<home>/apps/<id>/token`, written on first install | One installed application |

### `session.hello`

| Param | | |
|---|---|---|
| `token` | string, required | |
| `app` | string | The application id. Absent: authenticate as the owner. |
| `api` | integer | The protocol version the client speaks. Mismatch → `incompatible`. |

Result: `core_version`, `api`, `node`, `schema`, `session`, `principal` (`"owner"` or
`"app"`), and for an application `app`, `granted` and `pending` capabilities.

Unknown application and wrong token get the same answer, `unauthenticated`, after a short
delay. A second `hello` on the same connection is `invalid_request`.

## 3. Authority

Each method below lists who may call it:

- **owner** — only the owner principal.
- **app** — only an application (the owner has no document store of its own).
- **any** — any authenticated principal.
- **`<capability>`** — an application holding that capability, see
  [capabilities.md](capabilities.md). Where it says *owner or `<capability>`*, the owner
  may always call it.

Authority is checked on **every call** against the registry as it is at that moment. A
revoked capability fails on the next request; there is nothing to reconnect.

A refusal for a missing capability is `forbidden` with `data.capability` naming it, so an
application can tell the person exactly what to allow. Refusals are written to the audit
log.

## 4. Errors

`error.code` is the JSON-RPC number; `error.data.code` is the **stable** string an
application should switch on. `error.message` is English for a developer's log — an
application shows its own words, in its own language (madde 40).

| `data.code` | `code` | Meaning |
|---|---|---|
| `parse_error` | -32700 | Not JSON |
| `invalid_request` | -32600 | Not a JSON-RPC 2.0 request; a batch; a second hello |
| `method_not_found` | -32601 | No such method — or the called application does not provide it |
| `invalid_params` | -32602 | Missing or malformed parameter |
| `internal` | -32603 | The core failed; details are in its log, not here |
| `unauthenticated` | -32001 | No hello yet, or wrong credentials |
| `forbidden` | -32002 | Not the owner, or a capability not granted (`data.capability`) |
| `not_found` | -32003 | No such thing |
| `conflict` | -32004 | Stale revision (`data.rev` is the current one); already running |
| `app_unavailable` | -32005 | The called application is not installed, not running, or went away mid-call (`data.app`) |
| `timeout` | -32006 | The called application did not answer in time |
| `rate_limited` | -32007 | Too many notifications |
| `incompatible` | -32008 | Protocol, manifest format, downgrade, or database schema mismatch |
| `limit` | -32009 | Too large or too many |
| `remote_error` | -32010 | The called application answered with an error; it is in `data.error` |

## 5. Methods

### Core

| Method | Who | Params → result |
|---|---|---|
| `core.info` | anyone, even before hello | → `core_version`, `api`, `node`, `schema` |
| `session.ping` | any | → `pong` (ms) |
| `core.shutdown` | owner | → `stopping`. Launched applications are stopped; every session gets `shutdown`. |

### Applications

| Method | Who | Params → result |
|---|---|---|
| `apps.install` | owner | `manifest` ([schema](../schemas/app-manifest.v0.schema.json)), `path`?, `grant`? (list) → the registry entry, `token_file`, `upgraded_from`. Re-installing a newer version upgrades: data and token kept, grants kept only if still requested. An older version is `incompatible`. |
| `apps.uninstall` | owner | `app` → removes the application, **its documents, settings, search entries, jobs and notifications**, its token; closes its connections |
| `apps.grant` / `apps.revoke` | owner | `app`, `capabilities` → the registry entry. Only requested capabilities can be granted. |
| `apps.list` | owner | → `apps`: entries with `granted`, `pending`, `connected`, `process` |
| `apps.available` | any | → `apps`: `id`, `name`, `version`, `provides`, `publishes`, `connected` — what an application may integrate with, without grants or paths |
| `apps.launch` | owner | `app` → `pid`. Needs `entry` in the manifest. |
| `apps.stop` | owner | `app` → `stopped`, `forced`. The application gets `shutdown` and three seconds. |
| `apps.status` | owner | `app` → `connected`, `process` (`state`: `running`, `stopped`, `crashed`, `restarting`, `failed`; `restarts`; `last_code`) |
| `apps.call` | owner or `call:<app>` | `app`, `method`, `params`?, `timeout_ms`? (default 10 000, max 60 000) → whatever the provider answered |

**Calls between applications.** `apps.call` is forwarded to one connection of the target
as a request:

```json
{"jsonrpc":"2.0","id":"i42","method":"invoke","params":{"from":"org.eon.office.notes","method":"task.create","params":{"title":"Buy milk"}}}
```

The target answers `{"jsonrpc":"2.0","id":"i42","result":…}` (or `error`). The method must
be in the target's `provides`, or the core answers `method_not_found` without disturbing
it. If the target is not connected, or disconnects before answering, the caller gets
`app_unavailable` immediately — **an integration must treat this as normal**, because
applications are installed and running independently (madde 43b).

### Identity and directory

| Method | Who | Params → result |
|---|---|---|
| `identity.me` | any | → `principal`, `app`?, `user` (the owner's profile) |
| `profile.update` | owner | `display_name`?, `locale`? → `user` |
| `directory.get` | owner or `directory.read` | → `users`, `orgs`, `workspaces`, `teams`, `memberships` |
| `directory.create_user` | owner | `name` (a-z 0-9 . _ -), `display_name`? → `user` |
| `directory.create_org` | owner | `name` → `id`, `name`. The owner becomes its `owner`. |
| `directory.create_workspace` | owner | `name`, `org`? → `id`. Without `org`, the owner becomes its `owner`. |
| `directory.create_team` | owner | `workspace`, `name` → `id` |
| `directory.add_member` | owner | `user`, `kind` (`org`/`workspace`/`team`), `scope`, `role` (`guest`/`member`/`admin`/`owner`) → `added` |
| `directory.remove_member` | owner | `user`, `kind`, `scope` → `removed` |
| `access.check` | owner or `directory.read` | `user`, `permission`, `kind`, `scope` → `allowed`, `role`, `via` |

**Roles and permissions.** A role is held *in a scope* and reaches downwards: organisation
→ its workspaces → their teams. The strongest role on the path decides.

| Permission | guest | member | admin | owner |
|---|:-:|:-:|:-:|:-:|
| `read` | ✓ | ✓ | ✓ | ✓ |
| `write`, `share` | | ✓ | ✓ | ✓ |
| `delete`, `manage_members`, `manage_settings` | | | ✓ | ✓ |
| `administer` (delete the scope, appoint owners) | | | | ✓ |

### Storage

All `storage` — each application sees only its own documents.

| Method | Params → result |
|---|---|
| `storage.put` | `collection`, `id`, `doc` (any JSON but `null`, ≤ 512 KiB), `rev`? → `rev`, `hlc`. With `rev`, the write is refused as `conflict` unless it matches the current revision; `rev: 0` means "must not exist". |
| `storage.get` | `collection`, `id` → `doc`, `rev`, `hlc`, or `null` |
| `storage.delete` | `collection`, `id`, `rev`? → `rev`, `hlc`. Leaves a tombstone so the deletion syncs. |
| `storage.list` | `collection`, `after`?, `limit`? (≤ 1000) → `items` ordered by id |
| `storage.conflicts` | `limit`? → `items`: versions that lost a sync merge |
| `storage.resolve` | `conflict` → dismisses one, after the application has merged it |

Names (`collection`, `id`) are 1–128 characters of `A-Z a-z 0-9 . _ : -`.

### Sync

Owner only. See [sync.md](sync.md).

| Method | Params → result |
|---|---|
| `sync.export` | `since`? (a stamp), `limit`? (≤ 10 000) → `node`, `changes`, `until`, `complete` |
| `sync.import` | `changes` (≤ 10 000), `node`? → `applied`, `unchanged`, `conflicts`. Each affected application's sessions get `core/storage.changed`. |
| `sync.conflicts` | `app`?, `limit`? → `items` |

### Events

| Method | Who | Params → result |
|---|---|---|
| `events.publish` | `events.publish` | `event` (must be in the manifest's `publishes`), `payload`? (≤ 64 KiB) → `published` (the topic `<app>/<event>`) |
| `events.subscribe` | owner or `events.subscribe` | `pattern` → `subscriptions` |
| `events.unsubscribe` | owner or `events.subscribe` | `pattern` → `subscriptions` |

Patterns: `*`, `<namespace>/*`, or `<namespace>/<event>`, where the namespace is an
application id or `core`. At most 64 per connection; they end with the connection.
Delivered as:

```json
{"jsonrpc":"2.0","method":"event","params":{"topic":"org.eon.office.notes/note.created","from":"org.eon.office.notes","payload":{…},"at":1763539200000}}
```

Delivery is **at most once**. A connection that stops reading loses its own events once
its queue (1,024 messages) is full; the publisher never waits.

**Core events:**

| Topic | To | Payload |
|---|---|---|
| `core/app.installed`, `core/app.uninstalled` | subscribers | `app`, `version`? |
| `core/app.connected`, `core/app.disconnected` | subscribers | `app` |
| `core/app.started`, `core/app.stopped`, `core/app.crashed` | subscribers | `app`, `pid` or `code` |
| `core/notification.posted` | **owner** subscribers only | the notification |
| `core/prefs.changed` | **every** session, subscribed or not | `key`, `value` |
| `core/storage.changed` | the owning application's sessions | `collection`, `id` |

### Notifications

| Method | Who | Params → result |
|---|---|---|
| `notifications.post` | `notifications` | `title` (1–200), `body`? (≤ 2000), `urgency`? (`low`/`normal`/`high`) → the notification. At most **30 a minute** per application. |
| `notifications.list` | owner (all) or `notifications` (own) | `unread_only`?, `limit`? → `items`, newest first |
| `notifications.mark_read` | owner (any) or `notifications` (own) | `id` |

### Search

| Method | Who | Params → result |
|---|---|---|
| `search.index` | `search.index` | `id`, `title`?, `text`? (≤ 256 KiB), `visibility`? (`private` default, or `shared`), `workspace`? |
| `search.remove` | `search.index` | `id` → `removed` |
| `search.query` | owner or `search.query` | `query`, `limit`? → `hits`: `app`, `id`, `title`, `snippet`, `workspace`, `score` |

An application finds its own entries and every application's `shared` entries; the owner
finds everything. Every query term must match, as a prefix. Matching folds case and
diacritics the Turkish-safe way: `İ I ı i` are one letter, `ş→s`, `ğ→g`, `ç→c`, `ö→o`,
`ü→u` (madde 40). The index keeps a 160-character snippet, not the text.

### Background jobs

| Method | Who | Params → result |
|---|---|---|
| `schedule.add` | `schedule` | `name`, `at`? (ms) or `delay_ms`?, `every_ms`? (≥ 1000), `payload`? (≤ 16 KiB). Same name replaces. ≤ 100 per application. |
| `schedule.list` | `schedule` | → `jobs` |
| `schedule.cancel` | `schedule` | `name` |

When a job falls due the core sends `{"method":"job.due","params":{"name","payload","scheduled_at"}}`
to one connection of the application. If none is open, the job waits. A repeating job that
missed several runs is delivered **once**.

### Settings, preferences, themes

| Method | Who | Params → result |
|---|---|---|
| `settings.get` / `settings.set` / `settings.list` | any (own scope) | `key`, `value` (`null` removes; ≤ 64 KiB; ≤ 1000 keys) |
| `prefs.get` | any | → `locale`, `theme`, `reduce_motion`, `high_contrast`, `font_scale` |
| `prefs.set` | owner | `key`, `value`. Validated; every session receives `core/prefs.changed`. |
| `themes.list` | any | → `builtin`, `themes` |
| `themes.put` | owner | `theme` ([schema](../schemas/theme.v0.schema.json)) → `id`, `ratios`. Refused below WCAG AA. |

An application is expected to follow the person's preferences: `reduce_motion` means no
non-essential animation, `font_scale` scales text, `high_contrast` and `theme` choose
colours (madde 35).

### Audit and backup

| Method | Who | Params → result |
|---|---|---|
| `audit.list` | owner | `after`?, `limit`? → `entries`: `seq`, `at`, `actor`, `action`, `target`, `detail`, `hash` |
| `audit.verify` | owner | → `ok`, `entries`, `head` or `broken_at` |
| `backup.create` | owner | `path` (absolute, must not exist) → a checked copy of the database |

### Well-known methods

Names an application may put in `provides` so that a shell (EON Launcher) or another
application can rely on them without knowing the application. They are conventions on top
of `apps.call`, not core features; the core treats them like any other provided method.

| Method | Params → result | Meaning |
|---|---|---|
| `open` | `id` (an id the application put in the search index, or a document id) → anything | Bring the item to the person: show the application's window on it. Called when a person picks a search result. |
| `focus` | → anything | Bring the application's window to the front. |

An application that does not provide them is still a complete application; a shell then
shows the item without opening it.

### Well-known events

Event names an application may put in `publishes` so a shell can build suite-wide views
without knowing the application.

| Event | Payload | Meaning |
|---|---|---|
| `activity` | `id` (as for `open`), `title`, `kind` (`created`, `opened`, `edited`, `shared`), optional `at` (ms) | The person did something with an item. A shell keeps the recent items and the activity feed from these. |

Rules for `activity`: publish what the person did, never what an item **contains** — a
title, not a body; the shell stores it locally and nowhere else (madde 31, 36). An
application publishes it only for the person's own actions, never for background work.
A shell receives it by subscribing to `*` and keeping topics that end in `/activity`.

### Deep links

An `eon-office://` address points at an item in an application, for links between
documents, a notification's action, or the operating system's link handler:

```
eon-office://<app-id>/<item-id>[?<query>]
eon-office://org.eon.office.notes/n-2026-11-19
```

A shell that receives one checks that `<app-id>` is installed, starts it if it is
launchable and not running, and calls its `open` method with `{ "id": "<item-id>",
"query": "<query>" }` (percent-decoded). `<item-id>` uses the same characters as storage
ids. A link to an application that is not installed is shown as such, with the option to
find it in the marketplace; it is never resolved by guessing.

### Messages the core sends to applications

| Message | Kind | Application should |
|---|---|---|
| `event` | notification | handle or ignore |
| `invoke` | request | answer with the same `id` — within the caller's timeout |
| `job.due` | notification | do the work |
| `shutdown` | notification | save and exit; the core kills it after three seconds when it asked to stop it |
