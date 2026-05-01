# Architecture Decision Records

Index of load-bearing decisions in cyberdeck. Read these before changing
the security model, the storage layer, or the observability wiring.

| # | Title | Status |
|---|---|---|
| [0001](0001-forbid-unsafe-code.md) | Forbid `unsafe` code in the cyberdeck library | Accepted |
| [0002](0002-host-key-tofu-model.md) | Host key verification — TOFU with hard-fail on mismatch | Accepted |
| [0003](0003-credential-sanitization.md) | Credential sanitization on persistence | Accepted |

## Format

Each ADR follows:

- **Status** — Accepted / Superseded / Deprecated.
- **Context** — what forced the decision; relevant constraints.
- **Decision** — what we chose, with enough detail to point at the
  implementation.
- **Consequences** — what this enables, what it costs, what tradeoffs
  we explicitly accept.

New ADRs get the next sequential number. ADRs are append-only: a
superseded decision stays in the tree with its `Status` updated and a
note pointing at the replacement.
