---
layout: default
title: Bootstrap System
---

# Bootstrap System

[Home](index)

## Why Bootstrap Exists

Each homomorphic multiplication adds noise to a ciphertext. After a few multiplications, the noise overwhelms the plaintext and decryption returns garbage.

Bootstrap is the fix: it takes a noisy ciphertext and produces a fresh one with the same plaintext but reset noise. This is what makes "unlimited depth" possible.

## Depth Ceilings Without Bootstrap

Without bootstrap, each config has a hard ceiling:

| Config | Max Multiplications | Why |
|--------|-------------------|-----|
| secure_128 | 2 | Noise budget: 62,000mb. Each mul costs ~43,000mb. |
| secure_192 | 3 | Larger Q gives more headroom |

After this many multiplications, `decrypt` returns `NoiseExhausted` instead of a value.

## The Three Bootstrap Paths

### 1. Circular Bootstrap

```rust
let fresh = bootstrap.bootstrap(&noisy_ct, &bsk, &ksk)?;
```

The bootstrap secret key is a "lift" of the working secret key into the bootstrap ring. Simplest path. Uses `ClockworkBootstrap::bootstrap()`.

### 2. Non-Circular KSK Bootstrap

```rust
let fresh = bootstrap.bootstrap_with_ksk(&noisy_ct, &bsk, &ksk)?;
```

Uses an independent bootstrap secret key with a gadget key-switch key. More secure in theory (avoids circular security assumption).

### 3. Auto-Bootstrap (AutoBootstrapEvaluator)

```rust
let mut evaluator = AutoBootstrapEvaluator::new(&ctx, &bootstrap, &bsk, &ksk, &evk, &config);
let result = evaluator.mul_auto(&ct1, &ct2)?;  // bootstraps automatically if needed
```

Wraps `RNSFHEContext` and monitors noise budget. When budget drops below threshold (default 25%), bootstrap fires automatically. This is what delivers unlimited depth.

## How Bootstrap Works Internally

Three phases:

1. **ModSwitch Q to t**: Scale ciphertext from large modulus Q down to plaintext modulus t
2. **Homomorphic Inner Product**: Evaluate decryption circuit homomorphically using the bootstrap key (BSK)
3. **ModSwitch Q_boot to Q_work**: Scale result back to working modulus

The key insight: you're computing `decrypt(ct)` homomorphically --- without actually having the plaintext. The output is a fresh encryption of the same value.

## Key Generation for Bootstrap

```rust
use nine65::ops::bootstrap::ClockworkBootstrap;
use nine65::params::secure_configs::SecureConfig;

let config = SecureConfig::secure_128().into_config();
let ctx = RNSFHEContext::try_new(&config)?;
let mut rng = ShadowHarvester::with_seed(12345);

// Generate working keys
let work_keys = ctx.generate_keys_dual(&mut rng);

// Create bootstrap engine
let boot = ClockworkBootstrap::new(&config)?;

// Generate bootstrap keys (BSK + KSK)
let boot_keys = boot.generate_keys(&work_keys.secret_key, &mut rng)?;
```

## Auto-Bootstrap in Detail

The `AutoBootstrapEvaluator` in `ops/auto_bootstrap.rs`:

1. Performs the multiplication via `work_ctx.mul_dual_public()`
2. Attempts to consume the noise cost from the budget
3. If budget is exhausted (consume returns Err) OR budget falls below threshold: triggers bootstrap
4. Resets noise budget after bootstrap
5. Returns the fresh ciphertext

Key bug fix (Feb 25, 2026): The original code discarded the `Err` from `budget.consume()` with `let _ = ...`. When budget was exhausted, `remaining_mb` froze and never reached the trigger threshold, so bootstrap never fired. Fixed by checking the return value explicitly.

## Depth Ceiling Investigation Results

| Track | Question | Answer |
|-------|----------|--------|
| A | Symmetric ceiling for secure_128 | Depth 2 (noise exhausted at depth 3) |
| A | Symmetric ceiling for secure_192 | Depth 3 (noise exhausted at depth 4) |
| B | Does auto-bootstrap deliver unlimited depth? | Yes --- 100 muls, 99 bootstraps, zero drift |
| C | Does bootstrap accumulate directional bias? | No --- 50 rounds, no drift detected |

## Testing Bootstrap

```bash
# All bootstrap tests
cargo test -p nine65 --lib --release -- bootstrap --nocapture

# Just auto-bootstrap
cargo test -p nine65 --lib --release -- test_auto_bootstrap_chained_muls --nocapture

# Bootstrap integration tests
cargo test -p nine65 --test bootstrap_integration --release --nocapture

# Extreme bootstrap adversarial tests
cargo test -p nine65-extreme-tests --features extreme-tests --release -- bootstrap_adversarial --nocapture

# Depth stress tests (ceiling investigation)
cargo test -p nine65-extreme-tests --features extreme-tests --release -- depth_stress --nocapture
```
