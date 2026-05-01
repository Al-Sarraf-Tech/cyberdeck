# ADR 2: Host key verification — TOFU with hard-fail on mismatch

## Status

Accepted.

## Context

When cyberdeck opens an SSH session to a remote host, it must decide
how to verify the host key the remote presents during the handshake.
Three positions exist on the spectrum:

1. **Strict pinning** — refuse to connect to any host whose key is not
   already in `known_hosts`. The operator must populate `known_hosts`
   out-of-band before the first connection.
2. **TOFU (Trust On First Use)** — accept the key on the first
   connection (and optionally pin it), then refuse on subsequent
   connections if the key changes. This is what OpenSSH does by
   default with `StrictHostKeyChecking=ask`.
3. **No verification** — accept any host key and connect anyway. This
   is what `StrictHostKeyChecking=no` does and is unsafe by default.

cyberdeck targets a usability profile similar to OpenSSH from the
shell: an operator should be able to import targets from
`~/.ssh/config`, hit a key in the TUI, and connect without a manual
pre-flight step. Strict pinning is therefore a usability blocker on
first connections.

We also operate in a domain (key management for SSH targets) where the
single most catastrophic failure is silently authenticating to a MITM
that then captures key material as the operator exchanges it.

## Decision

cyberdeck uses TOFU with **hard-fail** on mismatch. Implementation in
`src/ssh_ops.rs::verify_host_key`:

- Match against existing `known_hosts` entry → proceed.
- Mismatch against existing entry → bail with an explicit error
  message naming the file to inspect, **and** emit a structured
  `error`-level tracing event with `op = "host_key_verify",
  outcome = "mismatch"`. No prompt, no override flag.
- Host not in `known_hosts` → proceed under TOFU. We log this at
  `info` level with `outcome = "tofu_unknown"` so an operator can
  audit first-touch events afterwards.
- `~/.ssh/known_hosts` does not exist → proceed (logged as
  `outcome = "tofu_no_file"`). This handles a fresh install gracefully.

Critically, we do **not** modify `known_hosts` ourselves. We delegate
that responsibility to OpenSSH (which the operator will use anyway for
shell access) so that cyberdeck never silently pins a key the operator
has not also seen via `ssh`.

## Consequences

Positive:

- First connections to a new host succeed without setup friction.
- A subsequent mismatch — the primary signal of an attacker
  interposing — produces a hard error and a structured log entry that
  is greppable for incident response.
- Operators retain a single source of truth (`known_hosts`) for which
  hosts are pinned; cyberdeck and OpenSSH agree.

Negative / accepted tradeoffs:

- A first-touch MITM is undetectable: if an attacker is on path the
  very first time cyberdeck connects, the attacker's key is what gets
  pinned (by the next `ssh` invocation, not by cyberdeck itself). This
  is the universal weakness of TOFU and we accept it.
- Operators rebuilding a host (legitimate key rotation) must manually
  `ssh-keygen -R <host>` to clear the stale entry. The runbook
  documents this.
- We do not currently surface the offered host key fingerprint in the
  CLI when a TOFU acceptance happens. A future enhancement could print
  it for operator confirmation; until then the JSON log line is the
  audit trail.
