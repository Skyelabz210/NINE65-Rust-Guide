---
layout: default
title: Extreme Tests
parent: Tools & Testing
nav_order: 4
---

# Extreme Tests


## What This Is

A separate crate (`nine65-extreme-tests`) containing 85 adversarial and boundary tests. These answer questions the normal 721-test suite does not ask. They're opt-in to avoid slowing down routine `cargo test` runs.

## Running

```bash
# All extreme tests
cargo test -p nine65-extreme-tests --features extreme-tests --release

# With output
cargo test -p nine65-extreme-tests --features extreme-tests --release -- --nocapture

# Specific module
cargo test -p nine65-extreme-tests --features extreme-tests --release -- k_elimination_extremes
cargo test -p nine65-extreme-tests --features extreme-tests --release -- bootstrap_adversarial
cargo test -p nine65-extreme-tests --features extreme-tests --release -- depth_stress
cargo test -p nine65-extreme-tests --features extreme-tests --release -- boundary_tests
```

## Modules

### noise_model_validation
Does actual noise growth ever exceed the estimated model? Is noise accumulation deterministic?

### k_elimination_extremes
What happens at K-Elimination's mathematical boundaries? Capacity near u128::MAX, maximum valid values, overflow detection.

### rns_boundary_tests
Does RNS reconstruction work for all configs? Partial product overflow for secure_192/256? U256 overflow detection in release builds.

### bootstrap_adversarial
Bootstrap on fresh ciphertext (no prior operations). Mismatched polynomial degree. Exhausted prime list. Maximum noise bootstrap. 100-deep chain. All three paths producing same plaintext.

### concurrent_operations
Thread safety under concurrent encryption, addition, multiplication, and bootstrap.

### entropy_extremes
Counter rollover. Determinism under load. Poisoned mutex recovery. Adaptive threading stability.

### montgomery_timing_audit
Empirical constant-time verification. Timing independence across input patterns. Boundary value correctness.

### ema_numerical_stability
EMA calculator with i64::MAX/MIN inputs. Alternating extremes. Alpha boundary values.

### formal_spec_linkage
Does the Rust implementation match the Coq/Lean4 formal proofs? Test vector generation from formal predicates.

### cross_config_operations
Wrong-config key mixups produce errors (not panics or UB). Encrypt with one config, decrypt with another.

### depth_stress_tests
Actual maximum depth per config without bootstrap. Unlimited depth validation via auto-bootstrap (100+ muls). Bootstrap accumulation bias hypothesis testing.

### negative_plaintext_coverage
Negative plaintexts handled correctly for all configs. Add negative + positive. Multiply negative by positive.

### key_serialization_roundtrip
Key serialization/deserialization lossless for all key types.

### boundary_tests
Capacity proximity detection. Post-switch margin verification. Safe/Warn/Critical region classification.

## Questions Answered

| # | Question | Module | Result |
|---|----------|--------|--------|
| Q1 | Does noise exceed estimates? | noise_model_validation | No |
| Q2 | Is noise deterministic? | noise_model_validation | Yes |
| Q3 | secure_128 depth without bootstrap? | depth_stress | 2 muls |
| Q4 | K-Elim capacity overflow? | k_elimination_extremes | Fixed (checked_mul) |
| Q5 | U256 overflow in release? | rns_boundary | Fixed (assert not debug_assert) |
| Q6 | Negative plaintexts all configs? | negative_plaintext | Verified |
| Q7 | Bootstrap on fresh ciphertext? | bootstrap_adversarial | Works |
| Q14 | Cross-config key mixup? | cross_config | Error, not panic |
| Q17 | Auto-bootstrap at 100+ depth? | depth_stress | Confirmed unlimited |
