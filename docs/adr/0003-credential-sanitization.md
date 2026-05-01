# ADR 3: Credential sanitization on persistence

## Status

Accepted.

## Context

cyberdeck stores SSH target profiles in
`~/.config/cyberdeck/config.json` so that the TUI's Targets tab
remembers what the operator added across runs. Each profile carries an
`AuthMethod` enum which can be either:

- `Password { password: String }`, or
- `KeyFile { private_key: String, passphrase: Option<String> }`.

Both `password` and `passphrase` are sensitive. `private_key` (the
filesystem path to a key) is not, by itself, secret — but the contents
of the file at that path are.

Two classes of risk apply when those fields touch disk:

1. **Plain-text on operator's filesystem.** A config file with
   passwords in it is a credential dump waiting to be backed up,
   synced to a cloud drive, or copied into a bug report.
2. **Permission drift.** Even with secrets in the file, if file
   permissions are wider than `0600` the secrets leak to other local
   users.

We need a design that prevents both, defaults to the safest behaviour,
and does not require the operator to remember to do anything.

## Decision

`src/storage.rs::save_config_to_path` enforces three guarantees on
every save:

1. **Sanitization before serialisation.** A
   `sanitize_for_persistence` pass runs over a clone of the config
   before `serde_json::to_string_pretty` ever sees it. For each
   target:
   - `Password` variants have their `password` cleared to an empty
     string.
   - `KeyFile` variants have their `passphrase` set to `None`.
   The sanitised clone is what goes through serialisation; the
   in-memory `AppConfig` the caller still holds keeps the secrets so
   the running session can authenticate.
2. **Atomic write with permission tightening.** We write to a
   `.config.<pid>.tmp` file in the same directory, `chmod 0600` it,
   then `fs::rename` over the target. If the rename fails we remove
   the temp. Both the directory (`0700`) and the final file (`0600`)
   get explicit `set_permissions` calls; a `WARN` tracing event fires
   if either chmod fails so an operator can detect drift.
3. **Sanitization is non-optional.** There is no flag, env var, or
   alternate path that bypasses it. The only way to reintroduce
   secret persistence would be to delete the `sanitize_for_persistence`
   call, which a reviewer would catch.

The expected operator workflow is:

- Add a target in the TUI with credentials → use it for the session.
- Quit cyberdeck → password/passphrase fields are wiped from disk.
- Next run → cyberdeck loads the structurally-correct profile but
  with empty credentials, and prompts (in the TUI) before connecting.

## Consequences

Positive:

- The on-disk config is safe to back up, sync, or paste into a bug
  report.
- A worst-case file-permission drift exposes only host/user/port and
  identity-file paths, never raw credentials.
- The `credentials_stripped_on_save` test in `storage::tests`
  validates the guarantee holds end-to-end for both auth variants.

Negative / accepted tradeoffs:

- Operators must re-enter passwords every session. For password auth
  this is the primary cost; we mitigate by encouraging key-file auth
  in the README and audit findings.
- Passphrase-protected key files require the passphrase to be entered
  per session as well. Same mitigation: the audit recommends
  passphrase-less ed25519 keys when the private key file is itself
  protected by filesystem permissions and full-disk encryption.
- Sanitization is one-way: an operator who wants to share a config
  with a teammate over an out-of-band channel is intentionally
  unsupported. They must use a configuration management tool for that.
