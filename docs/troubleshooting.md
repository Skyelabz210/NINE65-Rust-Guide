---
layout: default
title: Troubleshooting
parent: Reference
nav_order: 2
---

# Troubleshooting


## Build Errors

### `standard_128_insecure` not found (E0599)

```
error[E0599]: no function or associated item named `standard_128_insecure` found
```

**Cause**: Insecure config methods are gated behind `#[cfg(any(test, debug_assertions, feature = "allow_insecure"))]`. They don't exist in release builds.

**Fix**: Use `SecureConfig::secure_128().into_config()` instead:
```rust
use nine65::params::secure_configs::SecureConfig;
let config = SecureConfig::secure_128().into_config();
```

### `light_rns` not found

**Cause**: Method was renamed to `light_rns_insecure()`.

**Fix**: Use `FHEConfig::light_rns_insecure()` (only available in test/debug builds).

### Profile warnings for exact_transcendentals

```
warning: profiles for the non root package will be ignored
```

**Cause**: `exact_transcendentals/Cargo.toml` has its own `[profile]` section. Cargo ignores per-crate profiles in a workspace.

**Impact**: None. Harmless warning.

## Test Failures

### fhe_demo `expect_eq("add", sum, a + b)` fails

**Cause**: The demo doesn't wrap `a + b` modulo `t`. When `a + b >= t`, BFV correctly wraps but the expected value doesn't. Cosmetic demo bug only.

**Impact**: Demo display issue. Core FHE is correct.

### NoiseExhausted after 2-3 multiplications

**Cause**: Normal behavior for symmetric mode without bootstrap. secure_128 exhausts at depth 2, secure_192 at depth 3.

**Fix**: Use `AutoBootstrapEvaluator` for deeper circuits:
```rust
let mut eval = AutoBootstrapEvaluator::new(&ctx, &boot, &bsk, &ksk, &evk, &config);
let result = eval.mul_auto(&ct1, &ct2)?;  // auto-bootstraps when needed
```

### Auto-bootstrap never triggers (0 bootstraps)

**Cause**: This was a bug fixed in commit `50f221f`. The `budget.consume()` error was being discarded.

**Verify**: Make sure you're on a commit after `50f221f`. Check `ops/auto_bootstrap.rs` line ~70 --- it should say:
```rust
let budget_exhausted = self.budget.consume(NoiseOpType::MulCt, mul_cost).is_err();
if budget_exhausted || self.budget.should_bootstrap(self.trigger_permille) {
```

Not the old version:
```rust
let _ = self.budget.consume(NoiseOpType::MulCt, mul_cost);
if self.budget.should_bootstrap(self.trigger_permille) {
```

## Runtime Issues

### Tests are extremely slow

**Cause**: Running in debug mode (no `--release` flag).

**Fix**: Always use `--release` for test runs:
```bash
cargo test -p nine65 --lib --release    # correct
cargo test -p nine65 --lib              # 10-50x slower
```

### 11 compiler warnings

**Cause**: Unused variables in `symmetric_bootstrap.rs` and `shadow_entropy_monitor.rs`. Expected and harmless.

**Impact**: None. Zero errors is what matters.

### `cargo miri` fails with "not available for stable"

**Cause**: Miri requires the nightly toolchain. See [Rust Toolchain](rust-toolchain).

**Fix**:
```bash
rustup toolchain install nightly
rustup +nightly component add miri
cargo +nightly miri test -p nine65 --lib -- specific_test_name
```

## Common Mistakes

### Using float literals

NINE65 forbids all floating-point. If you add `3.14` or `f64` anywhere, clippy (and eventually compilation) will reject it.

**Fix**: Use integer or rational arithmetic for everything.

### Forgetting --exclude flags

```bash
# Wrong --- will try to build Python/WASM bindings and fail
cargo build --release --workspace

# Right
cargo build --release --workspace --exclude nine65-python --exclude nine65-wasm
```

### Using insecure configs in production code

Test configs (`test_fast_insecure`, `test_medium_insecure`, `light_rns_insecure`) are blocked in release builds. This is intentional --- they have inadequate security for real use.

Always use `SecureConfig::secure_128()` or higher for anything beyond testing.
