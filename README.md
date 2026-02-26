# NINE65 Rust Guide

Personal reference guide to the NINE65 v7 FHE system.

Live at: https://skyelabz210.github.io/NINE65-Rust-Guide/

## Structure

**37 pages across 6 sections** | ~8,000 lines of documentation

### Foundation
- [Getting Started](docs/getting-started.md) — Your first FHE computation
- [Building & Running](docs/building.md) — Build commands, binaries, configs
- [Cargo Reference](docs/cargo-reference.md) — Every cargo command with NINE65 examples
- [Rust Toolchain](docs/rust-toolchain.md) — Channels, components, proxy binaries
- [Rust Patterns](docs/rust-patterns.md) — Rust features as they appear in this codebase
- [Feature Flags](docs/feature-flags.md) — What each flag does, when to combine
- [Glossary](docs/glossary.md) — 50 terms defined and cross-linked

### Using the System
- [Cookbook](docs/cookbook.md) — Code-first reference for every operation
- [Key Management](docs/key-management.md) — All key types, generation, zeroization
- [Security Configs](docs/security-configs.md) — Parameters, verification, HE Standard
- [Batch & Galois](docs/batch-and-galois.md) — Packing values, SIMD rotations
- [Neural Ops](docs/neural-ops.md) — Encrypted ML: ReLU, Sigmoid, Softmax
- [MANA Accelerator](docs/mana.md) — Lane-parallel FHE pipeline
- [Entropy & RNG](docs/entropy.md) — ShadowHarvester, SecureRng, threading
- [FHE Service](docs/fhe-service.md) — HTTP sessions, Cloud Run deployment

### How It Works
- [Architecture](docs/architecture.md) — System overview, three layers
- [BFV Scheme](docs/bfv-scheme.md) — BFV math, two encryption APIs
- [BFV Parameters](docs/bfv-params.md) — n, q, t, CBD explained
- [K-Elimination](docs/k-elimination.md) — Exact division, anchors, the 5-anchor fix
- [NTT](docs/ntt.md) — Two engines, coefficient vs evaluation domain
- [Montgomery](docs/montgomery.md) — Constant-time modular arithmetic
- [Noise Budget](docs/noise-budget.md) — Millibits tracking, thresholds
- [Bootstrap](docs/bootstrap.md) — Three paths, auto-bootstrap, depth ceilings
- [Three-Lock Bootstrap](docs/three-lock.md) — Shannon + Montgomery + Clockwork layers
- [GSO-FHE](docs/gso-fhe.md) — Basin collapse, depth management
- [Boundary System](docs/boundary.md) — Capacity proximity detection
- [Kiosk](docs/kiosk.md) — Self-destructing computation units
- [Circuit Compiler](docs/circuit-compiler.md) — Static noise analysis, DAG compilation
- [Crate Map](docs/crate-map.md) — Every crate and key file

### Tools & Testing
- [Testing](docs/testing.md) — Every test command, filtering, output
- [Benchmarks](docs/benchmarks.md) — Criterion benches, nine65_bench, baselines
- [Test Tools](docs/test-tools.md) — nextest, llvm-cov, miri
- [Extreme Tests](docs/extreme-tests.md) — Adversarial boundary harness
- [Clockwork Core](docs/clockwork-core.md) — Formal-spec RNS, invariants
- [Known Answer Tests](docs/kat.md) — Regression test vectors

### Reference
- [Error Reference](docs/error-reference.md) — Every Nine65Error variant explained
- [Troubleshooting](docs/troubleshooting.md) — Common errors and fixes
