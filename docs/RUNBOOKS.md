# cyberdeck Runbooks

Operator playbook for diagnosing and recovering from cyberdeck failures.
Each entry maps a symptom (error message, stderr line, or behaviour) to
probable cause, diagnostic steps, and recovery actions.

Structured JSON logs are written one object per line at
`<log_dir>/cyberdeck.log.YYYY-MM-DD`. The default `log_dir` resolution is:

1. `CYBERDECK_LOG_DIR` environment variable.
2. `<config_dir>/logs/` (where `config_dir` is `CYBERDECK_CONFIG_DIR` or
   `~/.config/cyberdeck/`).
3. `~/.cache/cyberdeck/logs/`.
4. `/tmp/cyberdeck-logs/` as a last-resort fallback.

Filter for warnings and errors:

```bash
jq 'select(.level == "ERROR" or .level == "WARN")' \
  ~/.cache/cyberdeck/logs/cyberdeck.log.$(date -I)
```

---

## Host Key Mismatch (suspected MITM)

**Symptom**

```
Error: HOST KEY VERIFICATION FAILED for <host>:<port>!
The remote host key does not match the key in /home/<user>/.ssh/known_hosts.
This could indicate a man-in-the-middle attack.
```

JSON:

```json
{"level":"ERROR","fields":{"op":"host_key_verify","outcome":"mismatch","host":"...","port":22,"known_hosts":"...","message":"HOST KEY MISMATCH — possible MITM"}}
```

**Cause** Two scenarios:

1. **Legitimate rotation** — the remote host's host key was regenerated
   (reinstall, container rebuild, deliberate rotation) and the local
   `known_hosts` entry is stale.
2. **Active MITM** — an attacker is intercepting the connection with a
   different host key. Treat this as the default assumption until ruled
   out.

**Diagnose**

- Out-of-band, ask the operator of the remote host for the current host
  key fingerprint:
  ```bash
  ssh-keygen -l -f /etc/ssh/ssh_host_ed25519_key.pub   # on the remote
  ```
- Compare it to what cyberdeck saw: pull the offered key from the JSON
  log line above and compare fingerprints.
- Inspect existing `known_hosts` entry:
  ```bash
  ssh-keygen -F <host> -f ~/.ssh/known_hosts
  ```

**Recovery**

- If rotation is confirmed legitimate, remove the stale entry and let TOFU
  re-pin on next connect:
  ```bash
  ssh-keygen -R <host>
  # or for non-default port
  ssh-keygen -R "[<host>]:<port>"
  ```
- If MITM is suspected, **do not** clear the entry. Investigate network
  path (DNS, routing, captive portal, proxy) before retrying. Rotate any
  credentials that may have been exposed by previous successful sessions.

---

## SSH Connection Refused / Timeout

**Symptom**

```
Error: failed connecting to <host>:<port>
Caused by: Connection refused (os error 111)
```

or

```
Error: failed connecting to <host>:<port>
Caused by: deadline has elapsed
```

JSON:

```json
{"level":"WARN","fields":{"op":"ssh_connect","phase":"tcp","host":"...","port":22,"error":"...","message":"TCP connect failed"}}
```

**Cause** Network reachability or the remote SSH daemon:

- `Connection refused` — daemon not listening on the port.
- `No route to host` — network unreachable.
- timeout — packet drop, firewall silently blocking, or wrong host.

**Diagnose**

```bash
ss -tnp 2>/dev/null | grep <host>            # local socket state
nc -zv <host> <port>                          # raw port reachability
ssh -v -o ConnectTimeout=5 <user>@<host> -p <port>  # OpenSSH verbose
```

If `nc` succeeds but cyberdeck times out, check the JSON log for the
`phase` field — `handshake` failures indicate a non-SSH service on the
port or a protocol-version mismatch.

**Recovery**

- Verify port and host are correct in cyberdeck (Targets tab in TUI, or
  `~/.config/cyberdeck/config.json`).
- Confirm the remote `sshd` is running and listening on the expected
  port. `sudo systemctl status sshd` on the remote.
- Check local and remote firewalls (`firewall-cmd --list-all`,
  `iptables -L`).

---

## libssh2 Errors (Handshake / Channel)

**Symptom**

```
Error: failed SSH handshake with <host>:<port>
Caused by: [-43] Failure establishing ssh session
```

or

```
Error: failed opening channel for <host>:<port>
Caused by: [-7] Channel allocation failure
```

JSON:

```json
{"level":"WARN","fields":{"op":"ssh_connect","phase":"handshake","host":"...","port":22,"error":"[-43] ..."}}
```

**Cause** libssh2-side failures, common subcases:

- `[-43] Failure establishing ssh session` — KEX algorithm mismatch
  (usually a very old or very locked-down remote sshd) or version
  banner mismatch.
- `[-7] Channel allocation failure` — too many concurrent channels, or
  the remote denied a new session for a policy reason (per-user
  channel cap).
- `[-16] Method not supported` — remote does not support any auth
  method cyberdeck offered.

**Diagnose**

- `ssh -vvv <user>@<host> -p <port>` to see the OpenSSH key exchange
  in full and compare offered/accepted algorithms.
- Check remote `sshd` config for `KexAlgorithms`, `Ciphers`, and
  `PasswordAuthentication` / `PubkeyAuthentication` toggles.
- `journalctl -u sshd -n 100` on the remote for auth denial reason.

**Recovery**

- Update the remote sshd to a current version, or relax KEX algorithm
  restrictions if appropriate.
- For `[-7]`, wait and retry, or ask the remote operator to raise the
  `MaxSessions` / `MaxStartups` limits.
- For `[-16]`, ensure cyberdeck is offering an auth method the remote
  accepts (key vs password) — the JSON log's `auth` field shows which
  cyberdeck used.

---

## Key Passphrase Failure on Import

**Symptom**

```
Error: ssh-keygen could not read private key: incorrect passphrase
```

JSON:

```json
{"level":"WARN","fields":{"op":"key_import","outcome":"ssh_keygen_failed","private_key":"...","message":"ssh-keygen rejected the private key (likely passphrase failure)"}}
```

**Cause** The passphrase supplied to `import-key` (CLI) or the TUI
import flow does not match the encryption on the private key.

**Recovery**

- Re-enter the passphrase carefully — leading/trailing whitespace
  matters and is not stripped.
- If the passphrase is genuinely lost, the key is unrecoverable.
  Generate a fresh ed25519 key (`g` in the Keys tab or
  `cyberdeck import-key`) and re-exchange to all hosts.

Note: cyberdeck never logs the passphrase, the private key contents, or
the raw `ssh-keygen` stderr (which can echo key material on some
versions). Only the file path and that the operation failed are
recorded.

---

## Key Auth Rejected by Remote

**Symptom**

```
Error: key auth failed for <host>:<port>
Caused by: [-18] Username/PublicKey combination invalid
```

JSON:

```json
{"level":"WARN","fields":{"op":"ssh_connect","phase":"auth","auth":"key_file","host":"...","user":"...","error":"[-18] ..."}}
```

**Cause**

- The public key is not in the remote `~/.ssh/authorized_keys`.
- File permissions on the remote `~/.ssh` (must be `0700`) or
  `authorized_keys` (must be `0600`) are too loose — sshd refuses to
  read them.
- The user account on the remote does not exist or login is disabled.

**Diagnose**

- On remote: `ls -la ~/.ssh/`, `sudo -u <user> cat ~/.ssh/authorized_keys`.
- Check sshd logs: `journalctl -u sshd | grep <user>`.

**Recovery**

- Re-run cyberdeck's exchange flow: `cyberdeck exchange --host ... --user ... --public-key ~/.ssh/id_ed25519.pub --key-file <bootstrap-key>`.
  The exchange command applies `chmod 700 ~/.ssh` and
  `chmod 600 ~/.ssh/authorized_keys` itself, so it is also a remediation
  for the permission scenario when bootstrap auth (e.g. password) is
  available.
- If no bootstrap auth is available, fix permissions out-of-band (e.g.
  via console) before retrying.

---

## Config File Permissions Drift

**Symptom**

```
{"level":"WARN","fields":{"op":"config_save","phase":"chmod_file","path":"...","error":"..."}}
```

or operator notices `~/.config/cyberdeck/config.json` is world-readable.

**Cause** cyberdeck always writes the config dir at `0700` and the file
at `0600` after each save. If these calls fail (read-only filesystem,
ACLs, restricted container mount) the warning above appears and the
file may keep wider permissions from a prior tool.

**Diagnose**

```bash
ls -la ~/.config/cyberdeck/
stat -c '%a %n' ~/.config/cyberdeck/ ~/.config/cyberdeck/config.json
```

**Recovery**

```bash
chmod 700 ~/.config/cyberdeck/
chmod 600 ~/.config/cyberdeck/config.json
```

If the warning recurs after a manual fix, the underlying filesystem is
preventing the chmod — investigate mount options, ACLs, or container
volume settings. The next `save_config` will overwrite passwords/
passphrases with empty strings (sanitization runs before write), so
secrets cannot leak even if permissions were briefly loose.

---

## Tracing log_dir Unwritable

**Symptom**

```
cyberdeck: log_dir <path> not writable, JSON logs disabled: <err>
```

**Cause** `mkdir -p` failed on the resolved log directory. Common
causes: permission denied, the path exists as a regular file, parent
directory missing.

**Diagnose**

```bash
ls -la $(dirname <path>)
echo "$CYBERDECK_LOG_DIR"
```

**Recovery**

- Override to a writable path: `export CYBERDECK_LOG_DIR=/tmp/cd-logs`.
- Or fix the default location: `mkdir -p ~/.cache/cyberdeck/logs && chmod 700 ~/.cache/cyberdeck/logs`.

cyberdeck continues to run; only the JSON log channel is disabled. CLI
output (stdout/stderr) and TUI in-memory logs are unaffected.

---

## Health Check Procedure

Run this manually after any infrastructure change (config edit, install
upgrade, mount change):

```bash
cyberdeck list-keys                                          # local key catalog
cyberdeck audit-keys                                         # local key health
ls -la ~/.config/cyberdeck/                                  # secure perms
ls ~/.cache/cyberdeck/logs/ 2>/dev/null || echo "no logs"   # log_dir writable
tail -5 ~/.cache/cyberdeck/logs/cyberdeck.log.$(date -I) 2>/dev/null | jq .
```

All five should succeed (or print an explicit, expected message). If
any fails, find the matching section above.
