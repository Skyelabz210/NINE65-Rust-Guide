---
layout: default
title: Security Configs
---

# Security Configurations

[Home](index)

## The Three Production Configs

| Config | n | Primes | log2(q) | Classical | Quantum | Hybrid |
|--------|---|--------|---------|-----------|---------|--------|
| `secure_128` | 4096 | 3 x 30-bit | 90 | 129 | 86 | 129 |
| `secure_192` | 16384 | 5 x ~30-bit | 147 | 374 | 213 | 318 |
| `secure_256` | 16384 | 6 x ~30-bit | 177 | 311 | 177 | 264 |

### What the numbers mean

- **n**: Ring dimension. Larger = more security but slower operations. Must be a power of 2.
- **Primes**: NTT-friendly primes used to represent the ciphertext modulus Q via RNS.
- **log2(q)**: Total bit-width of Q (product of all primes). Larger Q = more noise headroom = deeper circuits, but weakens security for fixed n.
- **Classical**: Bits of security against classical computers (BKZ lattice attack cost).
- **Quantum**: Bits of security against quantum computers (Grover speedup applied).
- **Hybrid**: Bits of security against meet-in-the-middle + BKZ hybrid attack (accounts for ternary secret).

### Security threshold

A config "meets" its claimed level if the binding security (minimum of classical and hybrid) is at or above the target. Under the conservative Core-SVP cost model, all three configs meet their claimed levels.

## Using Configs in Code

```rust
use nine65::params::secure_configs::SecureConfig;

// Production configs (always available)
let config = SecureConfig::secure_128().into_config();
let config = SecureConfig::secure_192().into_config();
let config = SecureConfig::secure_256().into_config();

// Deeper circuit variant (128-bit security, 4 primes, n=8192)
let config = SecureConfig::secure_128_deep().into_config();
```

### Test-only configs

These are gated behind `#[cfg(any(test, debug_assertions, feature = "allow_insecure"))]` and will not compile in release builds:

```rust
// Only available in test/debug builds
let config = SecureConfig::test_fast_insecure().into_config();   // n=1024, ~40 bits
let config = SecureConfig::test_medium_insecure().into_config(); // n=2048, ~80 bits
```

## Verifying Security

Run the built-in lattice estimator:

```bash
# Core-SVP (conservative)
cargo run -p nine65 --release --bin security_estimator_baseline

# MATZOV (aggressive/realistic)
cargo run -p nine65 --release --bin security_estimator_baseline -- --cost-model matzov
```

Output (2026-02-25 baseline):

```
Core-SVP:  secure_128=129, secure_192=318, secure_256=264
MATZOV:    secure_128=116, secure_192=286, secure_256=237
```

The estimator is in `crates/nine65/src/params/security_estimator.rs`. It uses:
- HE Standard v1.1 methodology
- Integer-only arithmetic (millibits precision)
- Ternary secret penalty for hybrid attacks
- Optimal MITM/BKZ split search
- Zero external dependencies

## HE Standard Compliance

The HE Standard v1.1 table defines maximum log2(q) for each (n, security level) pair:

| n | Max log2(q) for 128-bit | Max log2(q) for 192-bit | Max log2(q) for 256-bit |
|---|------------------------|------------------------|------------------------|
| 4096 | 109 | 75 | 58 |
| 8192 | 218 | 152 | 118 |
| 16384 | 438 | 305 | 237 |

Our configs:
- `secure_128`: log2(q)=90 at n=4096 --- well under the 109 limit
- `secure_192`: log2(q)=147 at n=16384 --- well under the 305 limit
- `secure_256`: log2(q)=177 at n=16384 --- under the 237 limit

## Parameter History

The parameters were hardened in stages:

1. **Initial (pre-Feb 20)**: secure_192 used n=8192, secure_256 used 7 primes
2. **Feb 20 release**: secure_192 upgraded to n=16384 (code), but old baseline doc not refreshed
3. **Feb 22 fix**: secure_256 dropped 7th prime (log2q 203 to 177), improving hybrid security from 226 to 264 bits
4. **Feb 25 baseline**: Fresh estimator run confirming all current parameters

The old baseline doc (`LATTICE_ESTIMATOR_BASELINE_2026-02-09.md`) was a stale snapshot. The current baseline (`2026-02-25`) is authoritative.
