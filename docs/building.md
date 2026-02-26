---
layout: default
title: Building & Running
parent: Foundation
nav_order: 2
---

# Building & Running


## Prerequisites

- Rust stable toolchain (currently 1.93.1)
- No other dependencies. No Python, npm, docker, or external C libraries needed.

Check your setup:

```bash
rustc --version    # should show 1.93.x or later
cargo --version    # should match
```

## Building

```bash
# Full workspace, release mode (~1-2 min first time, seconds after)
cargo build --release --workspace --exclude nine65-python --exclude nine65-wasm
```

The `--exclude` flags skip the Python and WASM binding crates which need additional toolchains.

Expected output: 11 warnings (all in `symmetric_bootstrap.rs`, harmless unused variables), 0 errors.

## Binaries

Four binaries live in `crates/nine65/src/bin/`:

### nine65_v7_demo --- Full FHE roundtrip

```bash
cargo run -p nine65 --release --bin nine65_v7_demo
```

Demonstrates: key generation, encryption, homomorphic add, homomorphic multiply, decryption. Proves the system works end-to-end.

### nine65_bench --- Operation timing

```bash
cargo run -p nine65 --release --bin nine65_bench
```

Prints per-operation timing across configs. This is where the performance baselines come from:

| Config | Encrypt | Add | Mul | Decrypt |
|--------|---------|-----|-----|---------|
| secure_128 | 23.56ms | 0.83ms | 152.13ms | 11.06ms |
| secure_192 | 61.59ms | 2.10ms | 459.02ms | 29.00ms |

### security_estimator_baseline --- Parameter security verification

```bash
# Conservative model (Core-SVP)
cargo run -p nine65 --release --bin security_estimator_baseline

# Aggressive model (MATZOV)
cargo run -p nine65 --release --bin security_estimator_baseline -- --cost-model matzov

# CSV output
cargo run -p nine65 --release --bin security_estimator_baseline -- --format csv
```

Uses NINE65's own integer-only lattice estimator. Zero external dependencies.

### fhe_demo --- Interactive config selection

```bash
cargo run -p nine65 --release --bin fhe_demo
cargo run -p nine65 --release --bin fhe_demo -- --config standard_128
```

## Config Names

When binaries accept a `--config` flag:

| Name | Maps to | Notes |
|------|---------|-------|
| `standard_128` | `SecureConfig::secure_128()` | Default, production safe |
| `high_192` | `SecureConfig::secure_128()` in release | Insecure 192 config only available with `--features allow_insecure` |

## Build Profiles

```bash
# Debug (fast compile, slow run, assertions enabled)
cargo build

# Release (slow compile, fast run, assertions stripped)
cargo build --release

# Release with debug info (for profiling)
cargo build --release --profile=release-with-debug
```

Always use `--release` for testing and benchmarking. Debug builds are 10-50x slower and will distort timing results.

## Feature Flags

| Flag | What it enables | Default? |
|------|----------------|----------|
| `ntt_fft` | FFT-based NTT | Yes |
| `parallel` | Rayon parallelism | No (MANA is canonical) |
| `clockwork` | GRO timing gates, bound tracking | No |
| `exact_rational` | NexGen rational bridge | No |
| `shadow-entropy` | CRT shadow entropy harvester | No |
| `adaptive-threading` | Entropy-based adaptive threads | No |
| `accelerated` | MANA + UNHAL integration | No |
| `deterministic_rng` | Reproducible test output | No |
| `allow_insecure` | Test-only configs (BLOCKED in release) | No |
| `serde` | Serialization support | No |

Enable features with:

```bash
cargo test -p nine65 --lib --release --features "clockwork,exact_rational"
```
