<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 EON contributors
-->

# Capabilities — v0

A capability is a permission an application **asks for** in its manifest and the owner
**grants**. Installing grants nothing unless the owner says so. A grant can only cover a
capability the manifest requested, so an application's maximum power is always readable in
its manifest. Grants are revocable at any time and take effect on the next call.

Some things need no capability because they touch nothing but the application itself:
its own settings, the person's preferences (read-only), the theme list, `identity.me`,
`apps.available`, `session.ping`.

| Capability | Lets the application | Wording a shell should show the person |
|---|---|---|
| `storage` | Keep its own documents in the core; see its own sync conflicts | *Keep its data on this device* |
| `notifications` | Post to the shared notification list; read and dismiss its own | *Send you notifications* |
| `search.index` | Put its own items into the shared search index, as private or shared | *Make its items searchable* |
| `search.query` | Search the index: its own items and items other applications **shared** | *Search items other apps have shared* |
| `events.publish` | Emit the events listed in its manifest's `publishes` | *Announce changes to other apps* |
| `events.subscribe` | Receive other applications' events and the core's application events | *Notice changes in other apps* |
| `schedule` | Be woken at a time or periodically | *Run tasks in the background* |
| `directory.read` | Read people, organisations, workspaces, teams and roles; ask `access.check` | *See your organisation's directory* |
| `call:<app-id>` | Call the methods `<app-id>` declares in `provides` | *Use \<App name\>* |

## Rules for adding a capability

A new capability changes what every shell must explain to a person, so it is a spec change
with a pull request here, stating:

1. what it lets an application do that no existing capability does;
2. what a person should be told when asked to grant it, in one line;
3. what the core enforces, and what it cannot.

Capabilities are coarse on purpose. A permission prompt that lists twenty fine-grained
rights is one people click through; the boundaries that matter here are *its own data*,
*the person's attention*, *other applications*, and *the organisation*.
