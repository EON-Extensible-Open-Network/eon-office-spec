<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 EON contributors
-->

# Capabilities — v0

A capability is something an application **declares** in its manifest. The launcher and
the marketplace show the list before an application is installed, in the wording below;
the person can **revoke** any of them later, and the library refuses a revoked or
undeclared capability on the next use. An application's maximum reach is therefore always
readable in its manifest.

**What this is and is not.** It is transparency, and a guard against an application doing
by mistake what it never said it would. It is **not** a sandbox: a program running as the
person can open the files itself. A real boundary arrives with sandboxed third-party code
(madde 4); until then the honest description is the one above.

Some things need no capability because they touch nothing but the application itself: its
own settings and the person's preferences and themes (read-only).

| Capability | Lets the application | Wording a launcher should show the person |
|---|---|---|
| `storage` | Keep its own documents; see its own sync conflicts | *Keep its data on this device* |
| `notifications` | Post notifications to the operating system | *Send you notifications* |
| `search.index` | Put its own items into the shared search index, as private or shared | *Make its items searchable* |
| `search.query` | Search its own items and items other applications **shared** | *Search items other apps have shared* |
| `events.publish` | Add its items to the person's recent list; publish summaries other applications may read | *Show its items in your recent list and to apps you allow* |
| `events.subscribe` | Reserved (no effect in the library form) | — |
| `schedule` | Set reminders while it runs; catch up at start | *Remind you at a time* |
| `directory.read` | Read people, organisations, workspaces, teams and roles | *See your organisation's directory* |
| `call:<app-id>` | Open `<app-id>`'s items through links, and read the summaries `<app-id>` publishes | *Use \<App name\>* |

## Rules for adding a capability

A new capability changes what every launcher must explain to a person, so it is a spec
change with a pull request here, stating:

1. what it lets an application do that no existing capability does;
2. what a person should be told, in one line;
3. what the library enforces, and what it cannot.

Capabilities are coarse on purpose. A permission prompt that lists twenty fine-grained
rights is one people click through; the boundaries that matter here are *its own data*,
*the person's attention*, *other applications*, and *the organisation*.
