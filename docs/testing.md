---
layout: default
title: Testing
---

# Testing

[Home](index)

## Test Commands

### Core library (721 tests, ~3.5 min)

```bash
cargo test -p nine65 --lib --release
```

This is the primary test command. Covers all arithmetic, FHE operations, bootstrap, noise budget, entropy, security, NTT, Montgomery, and K-Elimination.

### Full workspace (1,351 tests)

```bash
cargo test --release --workspace --exclude nine65-python --exclude nine65-wasm
```

Adds: clockwork-core (46), exact_transcendentals (143), nexgen_rational (95), fhe-service (22), mana (30), unhal (10).

### Extreme boundary tests (85 tests, opt-in)

```bash
cargo test -p nine65-extreme-tests --features extreme-tests --release
```

See [Extreme Tests](extreme-tests) for what these cover.

## Filtering Tests

The filter string matches against the full test path. Examples:

```bash
# All bootstrap tests
cargo test -p nine65 --lib --release -- bootstrap

# Just NTT
cargo test -p nine65 --lib --release -- ntt

# Just Montgomery
cargo test -p nine65 --lib --release -- montgomery

# Just noise budget
cargo test -p nine65 --lib --release -- noise::budget

# Just security estimator
cargo test -p nine65 --lib --release -- security_estimator

# Just K-Elimination
cargo test -p nine65 --lib --release -- k_elimination

# A specific test by name
cargo test -p nine65 --lib --release -- test_auto_bootstrap_chained_muls
```

## Seeing Output

### --nocapture

By default, Rust's test runner hides `println!` output from passing tests. You only see output from failures.

`--nocapture` shows everything:

```bash
cargo test -p nine65 --lib --release -- bootstrap --nocapture
```

### --test-threads=1

By default, tests run in parallel. Output from different tests interleaves and becomes unreadable. Force sequential execution:

```bash
cargo test -p nine65 --lib --release -- --nocapture --test-threads=1 bootstrap
```

This is slower but the output is ordered and readable.

### Combining both

The maximum-visibility command:

```bash
cargo test -p nine65 --lib --release -- --nocapture --test-threads=1
```

Shows every println from every test, one test at a time, in order.

## Listing Tests Without Running

```bash
# List all core tests
cargo test -p nine65 --lib --release -- --list

# List tests matching a filter
cargo test -p nine65 --lib --release -- --list bootstrap

# Count tests
cargo test -p nine65 --lib --release -- --list 2>&1 | grep "test$" | wc -l
```

## Integration Tests

Separate from lib tests, these live in `tests/` directories:

```bash
# Bootstrap integration tests
cargo test -p nine65 --test bootstrap_integration --release

# Bootstrap parameter exploration
cargo test -p nine65 --test bootstrap_parameter_exploration --release
```

## Per-Crate Testing

| Crate | Command | Test Count |
|-------|---------|-----------|
| nine65 | `cargo test -p nine65 --lib --release` | 721 |
| clockwork-core | `cargo test -p clockwork-core --release` | 46 |
| exact_transcendentals | `cargo test -p exact_transcendentals --release` | 143 |
| nexgen_rational | `cargo test -p nexgen_rational --release` | 95 |
| fhe-service | `cargo test -p fhe-service --release` | 22 |
| mana | `cargo test -p mana --release` | 30 |
| unhal | `cargo test -p unhal --release` | 10 |
| nine65-extreme-tests | `cargo test -p nine65-extreme-tests --features extreme-tests --release` | 85 |

## What the Tests Cover

### Arithmetic (~200 tests)
- Montgomery multiplication, reduction, negation, boundary values
- NTT forward/inverse transforms, pointwise operations
- K-Elimination: CRT reconstruction, capacity boundaries, anchor validation
- RNS: add, multiply, level management, overflow detection
- U256: wide multiplication, overflow assertions

### FHE Operations (~150 tests)
- Encrypt/decrypt roundtrip for all configs
- Homomorphic add (ct+ct, ct+pt)
- Homomorphic multiply (ct*ct with relinearization)
- Depth sweeps (how many muls before noise exhaustion)

### Bootstrap (~80 tests)
- Circular bootstrap (boot_sk = lift of work_sk)
- Non-circular KSK bootstrap (independent boot_sk)
- Auto-bootstrap (noise-triggered, unlimited depth)
- Chained operations (mul-boot-mul sequences)
- All three paths produce exact plaintext recovery

### Security (~60 tests)
- Lattice estimator: Core-SVP and MATZOV models
- HE Standard compliance for all configs
- Production safety validation (insecure configs rejected in release)
- Constant-time operation verification

### Noise Budget (~40 tests)
- Budget consumption tracking
- Bootstrap reset accuracy
- Rescaling refunds (negative costs)
- Budget exhaustion detection

### Entropy (~30 tests)
- Shadow harvester determinism
- CSPRNG seeding
- Adaptive threading stability
