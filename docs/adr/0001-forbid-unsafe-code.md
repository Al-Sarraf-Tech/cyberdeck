# ADR 1: Forbid `unsafe` code in the cyberdeck library

## Status

Accepted.

## Context

cyberdeck handles credentials, private keys, host-key verification, and
opens authenticated SSH sessions to remote machines. A memory safety
defect — buffer overrun, use-after-free, or aliasing violation — in any
of those code paths has direct security consequences: leaked private
key material, credential corruption, or arbitrary code execution within
the operator's terminal session.

The library already depends on `ssh2` (a Rust binding over libssh2)
and `ring`-style cryptographic primitives via OpenSSL through `ssh2`'s
`vendored-openssl` feature. Those FFI boundaries are auditable in the
upstream crates; we do not need to re-implement any of them.

## Decision

`src/lib.rs` declares `#![forbid(unsafe_code)]`. This is a hard lint:
any `unsafe` block, `unsafe fn`, or `unsafe impl` introduced anywhere in
the library tree (including modules added later) will fail compilation,
not just produce a warning.

We deliberately use `forbid` rather than `deny`:

- `deny` can be silenced with `#[allow(unsafe_code)]` at any inner
  scope.
- `forbid` cannot be locally relaxed without removing the crate-level
  attribute itself, which is a visible and reviewable change.

The `main.rs` binary inherits the same posture by construction: it
contains only CLI plumbing and dispatches into the library, so it has
no reason to introduce `unsafe` either. If a future need ever arises
(e.g. a platform-specific `mlock` for in-memory key buffers), we will
write a separate ADR justifying it and contain the `unsafe` to a
narrowly scoped module.

## Consequences

Positive:

- Memory-safety defects in cyberdeck-original code are statically
  impossible. Reviewer attention can focus on logic bugs and
  cryptographic protocol misuse.
- Supply-chain auditing scope is reduced — only third-party
  dependencies need `unsafe` review.
- Removes an entire class of CVE patterns from consideration in
  release notes.

Negative / accepted tradeoffs:

- We cannot do FFI ourselves; we are dependent on existing Rust
  bindings (`ssh2`, `dirs`, etc.) being maintained.
- Some optimisations (e.g. zero-copy slice tricks, raw memory locking
  for secret material) require workarounds or upstream patches.
- Adding `unsafe` later requires removing the `forbid` attribute,
  which is a high-visibility commit and triggers code review attention
  by design.
