---
layout: default
title: Home
nav_order: 1
permalink: /
---

# NINE65 Rust Guide

Personal reference for the NINE65 v7 FHE system. Everything needed to build, test, run, and understand the system.

[Getting Started](getting-started){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[Cookbook](cookbook){: .btn .fs-5 .mb-4 .mb-md-0 }

---

## Cheatsheet
{: .text-delta}

<div class="cheatsheet" markdown="1">

### Build

```bash
cargo build --release --workspace --exclude nine65-python --exclude nine65-wasm
```

### Test

```bash
cargo test -p nine65 --lib --release                                              # Core (721 tests, ~3.5 min)
cargo test --release --workspace --exclude nine65-python --exclude nine65-wasm     # Full workspace (1,351 tests)
cargo test -p nine65-extreme-tests --features extreme-tests --release             # Extreme boundary (85 tests)
cargo test -p nine65 --lib --release -- bootstrap                                 # Filter: just bootstrap
cargo test -p nine65 --lib --release -- ntt                                       # Filter: just NTT
cargo test -p nine65 --lib --release -- --nocapture                               # Show println output
cargo test -p nine65 --lib --release -- --nocapture --test-threads=1              # Sequential + output
```

### Run Binaries

```bash
cargo run -p nine65 --release --bin nine65_v7_demo                                # Full system demo (real FHE)
cargo run -p nine65 --release --bin nine65_v7_demo -- --seed 99                   # Demo with custom seed
cargo run -p nine65 --release --bin nine65_bench                                  # Benchmark harness (JSON)
cargo run -p nine65 --release --bin nine65_bench -- --config secure_192           # Bench specific config
cargo run -p nine65 --release --bin nine65_bench -- --max-depth 100 -o bench.json # Bench with output file
cargo run -p nine65 --release --bin security_estimator_baseline                   # Lattice estimator (Core-SVP)
cargo run -p nine65 --release --bin security_estimator_baseline -- --cost-model matzov  # MATZOV model
cargo run -p nine65 --release --bin security_estimator_baseline -- --format csv   # CSV output
cargo run -p nine65 --release --bin fhe_demo                                      # Interactive FHE demo
cargo run -p nine65 --release --bin fhe_demo -- --a 42 --b 7 --config standard_128
```

### Criterion Benchmarks

```bash
cargo bench -p nine65 --bench timing                                 # Per-operation latency
cargo bench -p nine65 --bench throughput                             # Operations/second
cargo bench -p nine65 --bench fhe_scaling                            # FFT scaling across N
cargo bench -p nine65 --bench adaptive_rayon --features parallel     # Adaptive threading
cargo bench -p nine65 --bench threading_comparison --features parallel  # Thread count sweep
cargo bench -p nine65 --bench nine65_vs_seal_comparison              # Rust vs SEAL comparison
```

### Quality

```bash
cargo clippy --workspace --exclude nine65-python --exclude nine65-wasm  # Lint (zero float enforced)
cargo fmt --all -- --check                                              # Format check
cargo fmt --all                                                         # Auto-format
cargo doc --open -p nine65                                              # Generate & open API docs
cargo tree -p nine65                                                    # Dependency tree
```

### Coverage & Safety

```bash
cargo llvm-cov -p nine65 --lib --release --html                               # HTML coverage report
cargo llvm-cov --release --workspace --exclude nine65-python --exclude nine65-wasm --html  # Full workspace
cargo +nightly miri test -p nine65 --lib -- montgomery::tests::test_montgomery_mul  # Memory safety check
```

### Integration Tests

```bash
cargo test -p nine65 --test bootstrap_integration --release             # Bootstrap integration
cargo test -p nine65 --test bootstrap_parameter_exploration --release   # Parameter exploration
```

</div>

---

## Pages
{: .text-delta}

### [Foundation](foundation)
[Getting Started](getting-started) | [Building & Running](building) | [Cargo Reference](cargo-reference) | [Rust Toolchain](rust-toolchain) | [Rust Patterns](rust-patterns) | [Feature Flags](feature-flags) | [Glossary](glossary)

### [Using the System](using-the-system)
[Cookbook](cookbook) | [Key Management](key-management) | [Security Configs](security-configs) | [Batch & Galois](batch-and-galois) | [Neural Ops](neural-ops) | [MANA Accelerator](mana) | [Entropy & RNG](entropy) | [FHE Service](fhe-service)

### [How It Works](how-it-works)
[Architecture](architecture) | [BFV Scheme](bfv-scheme) | [BFV Parameters](bfv-params) | [K-Elimination](k-elimination) | [NTT](ntt) | [Montgomery](montgomery) | [Noise Budget](noise-budget) | [Bootstrap](bootstrap) | [Three-Lock Bootstrap](three-lock) | [GSO-FHE](gso-fhe) | [Boundary System](boundary) | [Kiosk](kiosk) | [Circuit Compiler](circuit-compiler) | [Crate Map](crate-map)

### [Tools & Testing](tools-and-testing)
[Testing](testing) | [Benchmarks](benchmarks) | [Test Tools](test-tools) | [Extreme Tests](extreme-tests) | [Clockwork Core](clockwork-core) | [Known Answer Tests](kat)

### [Reference](reference)
[Error Reference](error-reference) | [Troubleshooting](troubleshooting)
