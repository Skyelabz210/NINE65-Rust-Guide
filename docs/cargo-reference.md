---
layout: default
title: Cargo Reference
parent: Foundation
nav_order: 3
---

# Cargo Reference

[Home](index)

Every cargo command relevant to NINE65 development, grouped by purpose. Commands shown with NINE65-specific flags and patterns.

The workspace excludes `nine65-python`, `nine65-wasm`, and `nine65-ffi` from the default member set. Most workspace-wide commands need `--exclude` flags for the Python and WASM crates since they require additional toolchains.

---

## Building

### cargo build

Compile the workspace.

```bash
# Full workspace, release (production) mode
cargo build --release --workspace --exclude nine65-python --exclude nine65-wasm

# Single crate only
cargo build -p nine65 --release

# Debug mode (faster compile, slower runtime, enables debug_assertions)
cargo build -p nine65
```

The `--exclude` flags are needed because `nine65-python` requires PyO3/maturin and `nine65-wasm` requires wasm-pack. Without them, the build will fail.

Release profile uses `opt-level = 3`, `lto = "fat"`, `codegen-units = 1`, and `panic = "abort"` (defined in the workspace `Cargo.toml`).

### cargo check

Type-check without producing binaries. Much faster than `build`.

```bash
cargo check --workspace --exclude nine65-python --exclude nine65-wasm
cargo check -p nine65
```

Use this for rapid iteration when you only care about compilation errors, not runtime.

### cargo clean

Remove all build artifacts.

```bash
cargo clean
```

The `target/` directory can grow to several gigabytes. Clean when disk space is tight or when you suspect stale artifacts.

---

## Testing

### cargo test

Run tests. This is the most-used command in NINE65 development.

```bash
# Core nine65 library tests (689+ tests, ~2-3 min release)
cargo test -p nine65 --lib --release

# Full workspace (all crates, ~1300+ tests)
cargo test --release --workspace --exclude nine65-python --exclude nine65-wasm

# Extreme boundary tests (separate crate, opt-in)
cargo test -p nine65-extreme-tests --features extreme-tests --release
```

#### Filter patterns

Cargo test accepts `--` followed by a filter string. Matches against the full test path.

```bash
# Bootstrap tests only
cargo test -p nine65 --lib --release -- bootstrap

# NTT tests only
cargo test -p nine65 --lib --release -- ntt

# K-Elimination tests
cargo test -p nine65 --lib --release -- k_elim

# Security estimator tests
cargo test -p nine65 --lib --release -- security::tests

# Noise budget tests
cargo test -p nine65 --lib --release -- noise::budget

# Kiosk architecture tests
cargo test -p nine65 --lib --release -- kiosk

# Single specific test
cargo test -p nine65 --lib --release -- test_secure_128_meets_claims
```

#### Output control

```bash
# Show println! output (normally captured)
cargo test -p nine65 --lib --release -- --nocapture

# Sequential execution + output (useful for debugging interleaved output)
cargo test -p nine65 --lib --release -- --nocapture --test-threads=1

# List test names without running them
cargo test -p nine65 --lib --release -- --list
```

#### Integration tests

Integration tests live in `crates/nine65/tests/`. They run as separate binaries.

```bash
# All integration tests
cargo test -p nine65 --release

# Specific integration test file
cargo test -p nine65 --test bootstrap_integration --release
cargo test -p nine65 --test bootstrap_parameter_exploration --release
cargo test -p nine65 --test security_integration --release
cargo test -p nine65 --test formalization_invariants --release
cargo test -p nine65 --test error_variant_coverage --release
cargo test -p nine65 --test clockwork_cross_validation --release
cargo test -p nine65 --test ntt_encoder_caching --release
cargo test -p nine65 --test random_encrypt --release
cargo test -p nine65 --test rational_bridge_proptest --release
cargo test -p nine65 --test anchor_drift_diagnostics --release
cargo test -p nine65 --test test_192_256_bootstrap --release
```

#### Feature-gated tests

Some tests require specific feature flags:

```bash
# Tests that use allow_insecure configs
cargo test -p nine65 --lib --release --features allow_insecure

# Clockwork timing gate tests
cargo test -p nine65 --lib --release --features clockwork

# Exact rational bridge tests
cargo test -p nine65 --lib --release --features exact_rational

# Serialization round-trip tests
cargo test -p nine65 --lib --release --features serde
```

### cargo bench

Run Criterion benchmarks. Six bench files exist in `crates/nine65/benches/`:

```bash
# FHE scaling benchmarks (encrypt, add, mul, decrypt across configs)
cargo bench -p nine65 --bench fhe_scaling

# Comparison benchmarks (NINE65 vs SEAL-equivalent parameter sets)
cargo bench -p nine65 --bench nine65_vs_seal_comparison

# Throughput benchmarks (requires --features parallel)
cargo bench -p nine65 --bench throughput --features parallel

# Adaptive Rayon benchmarks (requires --features parallel)
cargo bench -p nine65 --bench adaptive_rayon --features parallel

# Operation timing benchmarks (requires --features benchmarks)
cargo bench -p nine65 --bench timing --features benchmarks

# Threading comparison (in benches/ directory)
# Note: threading_comparison.rs exists but has no [[bench]] entry in Cargo.toml
```

Criterion writes HTML reports to `target/criterion/`. Open `target/criterion/report/index.html` after a run.

To filter within a bench file:

```bash
cargo bench -p nine65 --bench fhe_scaling -- "encrypt"
```

---

## Running

### cargo run

Run a binary from the workspace. Four binaries are defined in `crates/nine65/`:

```bash
# Full v7 system demo (default seed=42)
cargo run -p nine65 --release --bin nine65_v7_demo
cargo run -p nine65 --release --bin nine65_v7_demo -- --seed 12345
cargo run -p nine65 --release --bin nine65_v7_demo -- --os-seed

# Operation benchmarking tool (requires --features serde)
cargo run -p nine65 --release --bin nine65_bench --features serde

# BFV encrypt/decrypt demo with configurable plaintexts
cargo run -p nine65 --release --bin fhe_demo
cargo run -p nine65 --release --bin fhe_demo -- --a 100 --b 200 --config standard_128

# Security estimator baseline report
cargo run -p nine65 --release --bin security_estimator_baseline
```

The `fhe_demo` binary accepts `--a`, `--b`, `--config`, `--seed`, and `--os-seed` flags. The `--config` flag selects from `standard_128`, `high_192`, and (with `allow_insecure`) `light` and `he_standard_128`.

---

## Quality

### cargo clippy

Lint the codebase. The workspace enforces `#![forbid(unsafe_code)]` on the core `nine65` crate and `#![deny(clippy::float_arithmetic)]` on `clockwork-core` and `fhe-service`.

```bash
cargo clippy --workspace --exclude nine65-python --exclude nine65-wasm -- -D warnings
cargo clippy -p nine65 -- -D warnings
```

### cargo fmt

Format all Rust source files.

```bash
cargo fmt --all
cargo fmt --all -- --check    # Check without modifying (CI mode)
```

### cargo doc

Generate API documentation.

```bash
cargo doc -p nine65 --no-deps --open
cargo doc --workspace --exclude nine65-python --exclude nine65-wasm --no-deps
```

The `--no-deps` flag skips building docs for external dependencies, which saves significant time.

---

## Dependencies

### cargo add / cargo remove

Add or remove dependencies. Prefer adding to the `[workspace.dependencies]` table in the root `Cargo.toml` and referencing with `{ workspace = true }` in crate-level `Cargo.toml` files.

```bash
cargo add sha2 -p nine65        # Add to nine65 crate
cargo remove sha2 -p nine65     # Remove from nine65 crate
```

### cargo tree

Visualize the dependency graph.

```bash
cargo tree -p nine65              # Full tree for nine65
cargo tree -p nine65 -d           # Show only duplicated dependencies
cargo tree -p nine65 -i zeroize   # Invert: who depends on zeroize?
```

### cargo metadata

Machine-readable JSON of the entire dependency graph. Useful for scripting.

```bash
cargo metadata --format-version 1 | python3 -m json.tool | head -100
```

### cargo fetch

Download all dependencies without building.

```bash
cargo fetch
```

Useful for pre-populating the cache before going offline.

### cargo update

Update dependencies within semver-compatible ranges (respects `Cargo.lock`).

```bash
cargo update                   # Update everything
cargo update -p zeroize        # Update a single dependency
```

### cargo vendor

Copy all dependencies into a `vendor/` directory for offline or auditable builds.

```bash
cargo vendor
```

### cargo generate-lockfile

Regenerate `Cargo.lock` from scratch.

```bash
cargo generate-lockfile
```

---

## Package Management

### cargo install / cargo uninstall

Install or remove binaries from crates.io or local paths.

```bash
cargo install cargo-llvm-cov     # Install coverage tool
cargo install cargo-nextest      # Install faster test runner
cargo uninstall cargo-llvm-cov   # Remove it
```

### cargo search

Search crates.io.

```bash
cargo search "fhe"
cargo search "zeroize"
```

### cargo publish / cargo package

Package and publish a crate. NINE65 is proprietary so you will not publish, but `cargo package` is useful for verifying the crate builds cleanly in isolation.

```bash
cargo package -p nine65 --allow-dirty    # Dry run
```

---

## Information

### cargo version

```bash
cargo --version
cargo +nightly --version    # If nightly is installed
```

### cargo info

Display metadata about a dependency.

```bash
cargo info zeroize
cargo info thiserror
```

### cargo help

```bash
cargo help test
cargo help bench
```

### cargo locate-project

Print the path to `Cargo.toml`.

```bash
cargo locate-project
```

### cargo pkgid

Print the fully qualified package ID.

```bash
cargo pkgid nine65
```

---

## Compiler

### cargo rustc

Pass flags directly to `rustc` for a single crate.

```bash
# Show expanded macros for nine65
cargo rustc -p nine65 -- -Zunpretty=expanded 2>&1 | head -200

# Emit assembly
cargo rustc -p nine65 --release -- --emit=asm
```

### cargo rustdoc

Build docs with custom rustdoc flags.

```bash
cargo rustdoc -p nine65 -- --document-private-items
```

### cargo fix

Automatically apply compiler-suggested fixes.

```bash
cargo fix --workspace --exclude nine65-python --exclude nine65-wasm
```

---

## Installed Extras

These are not built into cargo but are commonly installed for NINE65 development.

### cargo llvm-cov

Code coverage via LLVM instrumentation.

```bash
cargo llvm-cov --workspace --exclude nine65-python --exclude nine65-wasm --html
# Open target/llvm-cov/html/index.html
```

### cargo miri

Interpret code under Miri to detect undefined behavior. Requires nightly.

```bash
cargo +nightly miri test -p nine65 --lib -- test_secure_128_meets_claims
```

Miri is slow but catches memory safety issues that tests alone miss. Use it selectively on critical paths.

### cargo nextest

A faster, more featureful test runner.

```bash
cargo nextest run -p nine65 --release
cargo nextest run -p nine65 --release -E 'test(bootstrap)'    # Filter expression
```

### cargo pretty-test

Prettier test output formatting.

```bash
cargo pretty-test -p nine65 --lib --release
```

---

## Where to go next

- [Testing](testing) --- test philosophy, what each test suite covers, and how to read test output
- [Feature Flags](feature-flags) --- the 11 feature flags and when to enable each one
- [Building & Running](building) --- quick-reference build commands and binary descriptions
