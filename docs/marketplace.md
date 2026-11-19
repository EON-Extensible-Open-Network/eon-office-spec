<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 EON contributors
-->

# The EON Office marketplace — v0 (UNSTABLE)

How applications are listed, found, installed and updated, independently of the library and
of every application. The index is data in a Git repository
([`eon-office-marketplace`](https://github.com/EON-Extensible-Open-Network/eon-office-marketplace));
nothing about it is hard-coded in a client, and an application never needs it: anything
installable from the marketplace is equally installable from its manifest file.

## Shape

- One file per application: `index/apps/<id>.json`, an **entry**
  ([schema](../schemas/marketplace-entry.v0.schema.json)).
- The index tooling builds `index.json`: `{ "index": 0, "generated": "<date>", "apps": [<entry>…] }`,
  sorted by id, deterministic (the same files always build the same bytes).
- Publishing is a pull request adding or changing one file. Review is the pull request;
  history is the audit log. No accounts, no backend (madde 8, madde 30 Faz L0).

## Checks the index makes beyond the schema

1. The file name is `<id>.json`.
2. Each release's `manifest` is a valid application manifest whose `id` equals the entry's
   and whose `version` equals the release's.
3. Releases are newest first and versions are unique.
4. Every `requires` names an application in the index, and the requirement graph has no
   cycle; `recommends` names applications in the index or is a warning.
5. Every URL is `https://`.
6. `official: true` only for sources under the EON organisation.
7. In CI, every artifact is downloaded once and its size and SHA-256 compared. An entry
   whose file changed after listing fails.

## What a client does

**Discover.** Read `index.json` from a URL the person configured (default: the project's,
once one is published). Show name, summary, publisher, `verified`, `official`, licence,
source, and — before anything else — the capabilities the release's manifest requests, in
the words of [capabilities.md](capabilities.md).

**Choose a release.** The newest release on the person's channel (`stable` unless they chose
`preview`) whose manifest `api` range contains the client's shared-store version, with an artifact
for this platform. None: say why (no build for this platform, needs a newer EON Office).

**Resolve requirements.** Each `requires` entry must be installed at `min` or newer, or be
installable from the index; install those first. `recommends` are offered, never installed
silently.

**Download and verify.** Download the artifact; compare size and SHA-256 with the entry —
a mismatch stops the install. Then the signature:

| Signature | Trust set | Client |
|---|---|---|
| valid, key trusted | — | installs |
| present, invalid or key unknown | — | **refuses** |
| absent | empty (today, for everyone) | installs only after the owner confirms an **unsigned** install, worded as such |
| absent | non-empty | refuses unless the owner overrides per install |

The trust set and envelope are EON Stream's (madde 34): Ed25519, JCS subject
`{"id","sha256","version"}`, keys held by a legal entity that does not exist yet — so the
set is empty and the honest state of every release is *unsigned*.

**Install.** A single-file artifact is saved in the client's folder for the application
as the manifest's `entry.command` plus the platform's executable suffix (`hello` →
`hello.exe` on Windows), which is where [links](shared-store.md#5-links) look for it. An
application with no `entry` is named after the last label of its id
(`org.eon.office.caller` → `caller`). Then the client starts the application once so it
registers itself with that folder as `path`. Nothing is revoked or granted beyond what the
manifest declares, and the person saw that list before downloading.

**Update.** A newer compatible release on the same channel is offered; installing it
replaces the file and starts it once; its data stays, and registration never downgrades.
Revocation lists (madde 34) apply per version once they exist.

**Remove.** The client unregisters the application (`Shared::unregister`) and removes its
program folder; the person chooses whether `apps/<id>/` — their data — goes too. The
marketplace is not involved.

## Not in v0, and why

| Feature | Status |
|---|---|
| Ratings and reviews | Need accounts and moderation, which is personal data and hosting obligations: **behind madde 30's Faz L1 gate**. The entry format leaves room; no client shows invented ratings. |
| Publisher verification | A manual process: the publisher proves control of the `source` repository and a contact address to an index maintainer, who sets `verified` in a separate commit. Self-verification in one's own pull request is refused by review. |
| Paid applications | Madde 10 and Faz L2. |
| Search ranking, featured lists | Need a backend (madde 8). Clients sort and filter `index.json` locally. |
