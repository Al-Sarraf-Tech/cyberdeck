# CLAUDE.md — cyberdeck

Ratatui-based TUI + CLI for SSH key management. Requires `rust-edition = "2024"`, `rust-version = "1.85"`.

## Quick Reference
```bash
cargo build
cargo run                       # launch TUI
cargo run -- list-keys
cargo run -- run --host <ip> --port 22 --user <user> --key-file ~/.ssh/id_ed25519 --cmd "whoami"
cargo test --all-targets
./scripts/regression.sh         # full Docker-based regression
./scripts/install-local.sh      # install binary locally
```

Config: `~/.config/cyberdeck/config.json`.

TUI keybindings: `1-4` switch tabs, `q` quit, `F2` theme, `g` generate key, `a` add target, `x` exchange key, `e`/Enter run SSH command.

## Build

```bash
cargo build --release
```

## Test

```bash
cargo test --workspace
./scripts/regression.sh     # full Docker-based regression
```

## Lint

```bash
cargo fmt --all --check
cargo clippy --workspace --all-targets -- -D warnings
```

## Rust CI Gate
```bash
cargo fmt --all --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```
Release profile: `codegen-units = 1`, `lto = true`, `strip = true`.

## CI/CD
- Org CI must pass before pushing to personal. Runners: `linux-mega-1`, `wsl2-runner`, `dominus-runner`.
