# cyberdeck

[![CI](https://github.com/Al-Sarraf-Tech/cyberdeck/actions/workflows/ci-rust.yml/badge.svg?branch=main)](https://github.com/Al-Sarraf-Tech/cyberdeck/actions/workflows/ci-rust.yml)
[![License: Apache-2.0](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)
[![Release](https://img.shields.io/github/v/release/Al-Sarraf-Tech/cyberdeck)](https://github.com/Al-Sarraf-Tech/cyberdeck/releases/latest)

> CI runs on self-hosted runners managed by the [Haskell Orchestrator](https://github.com/Al-Sarraf-Tech/Haskell-Orchestrator).

Terminal UI and CLI for SSH key management, key exchange, and remote command execution. Built with Rust and [Ratatui](https://ratatui.rs).

---

## Demo

<img width="2227" height="1288" alt="Cyberdeck TUI — Cyberpunk theme" src="https://github.com/user-attachments/assets/a37a1ea7-2bf5-498e-8fb0-05b448901f19" />

---

## Features

- **Key management** — list, generate (ed25519), and import SSH keys from `~/.ssh`
- **Target profiles** — store and manage SSH connection profiles (host, port, user, auth method)
- **Key exchange** — deploy a local public key to a remote host's `authorized_keys`
- **Remote commands** — run commands on targets from the built-in SSH console
- **SSH config import** — pull targets from `~/.ssh/config` automatically; wildcard and empty-hostname entries are skipped safely
- **Key health audit** — detect weak algorithms (DSA, RSA), old keys, and orphaned public keys with severity-ranked findings
- **Export** — print saved targets as ready-to-run `ssh` commands
- **Host key verification** — checks `~/.ssh/known_hosts` to prevent MITM attacks; hard-aborts on mismatch
- **Credential safety** — passwords and passphrases are stripped before any write to disk; memory-only for the session
- **Five themes** — Cyberpunk, Synth, Matrix, Ember, Glacier — switchable live with `F2`
- **No unsafe code** — enforced via `#![forbid(unsafe_code)]`

---

## Installation

### From a release binary

Download the latest binary for your platform from the [Releases](https://github.com/Al-Sarraf-Tech/cyberdeck/releases/tag/v0.1.9) page.

| Platform | Asset |
|---|---|
| Linux x86_64 | `cyberdeck-v0.1.9-x86_64-unknown-linux-gnu.tar.gz` |
| Linux ARM64 | `cyberdeck-v0.1.9-aarch64-unknown-linux-gnu.tar.gz` |
| Windows x86_64 | `cyberdeck-v0.1.9-x86_64-pc-windows-msvc.zip` |

Linux packages are also available: `.rpm` (Fedora/RHEL), `.deb` (Debian/Ubuntu), `.pkg.tar.zst` (Arch/Pacman).

### From source

Requires Rust 1.85+ (edition 2024).

```bash
git clone https://github.com/Al-Sarraf-Tech/cyberdeck.git
cd cyberdeck
cargo build --release
./target/release/cyberdeck
```

### Development install

```bash
./scripts/install-local.sh
cyberdeck
```

---

## Usage

Launch the TUI (default when no subcommand is given):

```bash
cyberdeck
```

### TUI tabs and keybindings

| Key | Action |
|---|---|
| `1` `2` `3` `4` | Switch to Keys / Targets / Exchange / Console tab |
| `Left` / `Right` | Cycle tabs |
| `Up` / `Down` | Navigate list items |
| `r` | Refresh current view |
| `F2` | Cycle theme forward |
| `q` | Quit |

**Keys tab**

| Key | Action |
|---|---|
| `g` | Generate a new ed25519 key |
| `i` | Import an existing private key |
| `h` | Run key health audit |

**Targets tab**

| Key | Action |
|---|---|
| `a` | Add a new target profile |
| `d` | Delete selected target |
| `t` | Test connection to selected target |
| `c` | Import targets from `~/.ssh/config` |

**Exchange tab**

| Key | Action |
|---|---|
| `x` | Exchange (deploy) a local public key to the selected target |
| `f` | Fetch remote `authorized_keys` entries |

**Console tab**

| Key | Action |
|---|---|
| `e` or `Enter` | Enter input mode |
| `r` | Run the typed command on the selected target |
| `Esc` | Leave input mode |
| `c` | Clear output |

### CLI subcommands

```bash
# List local SSH keys from ~/.ssh
cyberdeck list-keys

# Import a private key into the catalog
cyberdeck import-key --private-key ~/.ssh/id_ed25519

# Import targets from ~/.ssh/config
cyberdeck import-config

# Audit keys for security issues (algorithm, age, orphaned keys)
cyberdeck audit-keys

# Export saved targets as runnable ssh commands
cyberdeck export

# Run a remote command
cyberdeck run --host 10.0.0.12 --port 22 --user dev \
  --key-file ~/.ssh/id_ed25519 --cmd "uname -a"

# Exchange a public key to a remote host's authorized_keys
cyberdeck exchange --host 10.0.0.12 --port 22 --user dev \
  --key-file ~/.ssh/id_ed25519 --public-key ~/.ssh/id_ed25519.pub

# Fetch remote authorized_keys entries
cyberdeck fetch --host 10.0.0.12 --port 22 --user dev \
  --key-file ~/.ssh/id_ed25519
```

For SSH authentication, supply exactly one of:

- `--key-file <path> [--passphrase <passphrase>]`
- `--password <password>`

---

## Themes

Five built-in themes cycle with `F2` in the TUI or persist across sessions via config:

| Theme | Description |
|---|---|
| **Cyberpunk** (default) | Neon yellow and magenta on dark |
| **Synth** | Purple and cyan on deep blue |
| **Matrix** | Green-on-black monochrome |
| **Ember** | Warm orange and red tones |
| **Glacier** | Cool blue-white on dark slate |

The active theme is stored in `~/.config/cyberdeck/config.json` and restored on next launch.

---

## Configuration

Config file: `~/.config/cyberdeck/config.json`

The directory is created with `0700` permissions; the file with `0600` (owner-only). Passwords and passphrases are stripped from the config on every write — they exist only in memory for the current session.

Override the config directory with the `CYBERDECK_CONFIG_DIR` environment variable.

---

## Security

- **Host key verification** — remote host keys are checked against `~/.ssh/known_hosts` on every connection. A mismatch aborts with a MITM warning. Hosts not yet in `known_hosts` proceed under TOFU (Trust On First Use).
- **No secrets on disk** — passwords and passphrases are stripped before writing `config.json`. Session-only.
- **Config permissions** — `~/.config/cyberdeck/` created `0700`; `config.json` created `0600`.
- **Key name validation** — rejects path separators and null bytes to prevent path traversal.
- **SSH config import hardening** — wildcard patterns (`*`, `?`) and entries with empty or missing hostnames are skipped.
- **Port validation** — out-of-range port values are rejected at parse time before any connection attempt.
- **TUI layout safety** — popup rendering guards against zero-height terminals to prevent panics.
- **No unsafe code** — `#![forbid(unsafe_code)]` enforced crate-wide.

---

## Architecture

```
src/
  main.rs        CLI dispatch (clap) + TUI entry point
  lib.rs         Crate-level attributes and re-exports
  models.rs      Core data types: LocalKey, TargetProfile, AuthMethod, AppConfig, CommandResult
  keys.rs        Key scanning (scan_local_keys), ed25519 generation, private key import
  ssh_ops.rs     SSH operations via libssh2: host key verification, key exchange, remote commands, fetch
  storage.rs     JSON persistence with credential sanitization and strict file permissions
  health.rs      Key health audit: algorithm check, age check, orphan detection
  ssh_config.rs  ~/.ssh/config parsing and safe target import
  tui.rs         Ratatui TUI: four tabs, five themes, modal forms, live input
ci/orchestrator/ Haskell CI/CD orchestrator (workflow and packaging generation)
packaging/       Generated RPM, DEB, and PKGBUILD specs
scripts/         regression.sh (Docker SSH integration tests), install-local.sh, build-packages.sh
tests/           regression.rs integration test suite
```

**Key dependencies:**

| Crate | Role |
|---|---|
| `ratatui` 0.29 | TUI rendering |
| `crossterm` 0.28 | Terminal input/output |
| `ssh2` 0.9 (vendored OpenSSL) | SSH protocol |
| `clap` 4.5 | CLI argument parsing |
| `serde` / `serde_json` | Config serialization |

---

## Development

### CI gate (run before committing)

```bash
cargo fmt --all --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --all-targets
```

### Full regression (Docker-backed SSH server)

```bash
./scripts/regression.sh
```

### Haskell CI orchestrator

The `ci/orchestrator/` tool generates all GitHub Actions workflows and packaging specs. Edit via orchestrator, not by hand:

```bash
cabal run orchestrator -- ci        # local CI gate
cabal run orchestrator -- generate  # regenerate workflows
cabal run orchestrator -- release   # regenerate packaging files
cabal run orchestrator -- full      # all of the above
```

---

## Release flow

```bash
# 1. Run the full orchestrator pipeline
cabal run orchestrator -- full

# 2. Commit and push
git add -A && git commit -m "chore: prepare release"
git push origin main

# 3. Tag and push — triggers the release workflow
git tag -a v$(awk -F'"' '$1 ~ /^version = / {print $2; exit}' Cargo.toml) -m "Release"
git push origin --tags
```

The release workflow:

1. Verifies the tag version matches `Cargo.toml`
2. Runs the full CI gate and Docker regression
3. Runs `cargo audit`, `cargo deny`, and secrets scanning via gitleaks
4. Packages Linux binaries as RPM, DEB, and Pacman
5. Publishes a GitHub Release with all assets and SHA256 checksums

---

## CI/CD

This project is governed by the [Haskell Orchestrator](https://github.com/Al-Sarraf-Tech/Haskell-Orchestrator) — a Haskell-based multi-agent CI/CD framework for pre-push validation, code quality enforcement, and release management across the Al-Sarraf-Tech organization. Workflows run on self-hosted runners (`linux-mega-1`, `wsl2-runner`, `dominus-runner`).

---

## License

[Apache-2.0](LICENSE)
