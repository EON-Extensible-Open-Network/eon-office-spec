<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 EON contributors
-->

# Contributing to eon-office-spec

The organisation-wide guide —
[`.github/CONTRIBUTING.md`](https://github.com/EON-Extensible-Open-Network/.github/blob/main/CONTRIBUTING.md)
— applies: Code of Conduct, DCO sign-off (`git commit -s`), Conventional Commits with the
`spec` scope.

Particular to this repository:

- **A contract change comes with examples.** Add at least one file to `examples/valid` and
  one to `examples/invalid` that show the change, named `<manifest|message|theme>-*.json`.
- **Patterns must mean the same in every engine.** The validator is JavaScript (Ajv, Unicode
  mode) and the core is Rust; a regular expression one accepts and the other rejects has
  broken a release before. Keep to plain character classes; avoid escapes inside classes.
- **Say what it costs.** Who has to change, and what happens to applications that do not.
- Contributions are licensed Apache-2.0.
