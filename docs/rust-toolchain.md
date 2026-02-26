---
layout: default
title: Rust Toolchain
parent: Foundation
nav_order: 4
---

# Rust Toolchain


## What's Installed

NINE65 runs on **Rust stable**. Current version: `rustc 1.93.1`.

### Installed components

| Component | What it does |
|-----------|-------------|
| `cargo` | Build system and package manager |
| `rustc` | The Rust compiler |
| `clippy` | Linter --- catches common mistakes |
| `rustfmt` | Code formatter |
| `rust-docs` | Offline documentation |
| `rust-std` | Standard library |

### Proxy binaries (stubs, not installed)

| Binary | What it would be | Status |
|--------|-----------------|--------|
| `cargo-miri` | Memory safety checker | Stub --- needs nightly |
| `rls` | Old language server | Stub --- deprecated, use rust-analyzer |
| `rust-analyzer` | Language server for editors | Stub --- install via your editor |
| `rust-gdb` | GDB debugger integration | Stub --- install via `rustup component add` |
| `rust-gdbgui` | GDB web frontend | Stub --- same |
| `rust-lldb` | LLDB debugger integration | Stub --- same |

All binaries in `~/.cargo/bin/` are symlinks to `rustup`. The only real binary is `rustup` itself (20MB). When you run any tool, rustup checks the tool name, looks up whether the component is installed, and either forwards to the real tool or shows an error.

## Release Channels

| Channel | What it is | When to use |
|---------|-----------|-------------|
| **stable** | Production-ready, everything works | Always (your default) |
| **beta** | Next stable, being tested | Never needed for NINE65 |
| **nightly** | Bleeding edge, experimental features | Only for miri |

You only need stable. Nightly is optional, only if you want to run miri.

## Useful Commands

```bash
# Check current toolchain
rustup show

# Update to latest stable
rustup update stable

# Check what's installed
rustup component list --installed

# Install a component
rustup component add clippy

# Run clippy (linter)
cargo clippy --workspace --exclude nine65-python --exclude nine65-wasm

# Run rustfmt (formatter)
cargo fmt --all

# Check formatting without changing files
cargo fmt --all -- --check
```

## Additional Cargo Tools

These are installed separately via `cargo install` and live as real binaries in `~/.cargo/bin/`:

| Tool | Install | Purpose |
|------|---------|---------|
| cargo-nextest | `cargo install cargo-nextest` | Better test runner with timing |
| cargo-pretty-test | `cargo install cargo-pretty-test` | Tree-view test output |
| cargo-llvm-cov | `cargo install cargo-llvm-cov` | Code coverage reports |

See [Test Tools](test-tools) for usage details.

## The Target Triple

When you see `x86_64-unknown-linux-gnu`:

```
x86_64   -   unknown   -   linux   -   gnu
  |            |            |          |
 CPU arch    Vendor        OS        C library
```

`unknown` means "no specific vendor" (as opposed to `apple` for macOS or `pc` for Windows). This is normal for Linux systems. It has nothing to do with missing information --- it's just the naming convention.
