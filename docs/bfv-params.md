---
layout: default
title: BFV Parameters
parent: How It Works
nav_order: 3
---

# BFV Parameters

[Home](index)

Choosing FHE parameters involves balancing security, performance, and noise capacity. This page explains each parameter, the constraints that govern them, and the exact values NINE65 uses.

## The Parameters

### N --- Ring Dimension

The polynomial degree. Must be a power of 2 (required by the NTT). Larger N means more security, more noise headroom, and slower operations.

| N | Security Range | Typical Use |
|---|---------------|-------------|
| 1024 | ~40 bits | Testing only |
| 2048 | ~80 bits | Testing only |
| 4096 | 128 bits | Standard production |
| 8192 | 128 bits (deep) | Deeper circuits at 128-bit security |
| 16384 | 192--256 bits | High/maximum security |

### q --- Ciphertext Modulus

The product of all NTT-friendly primes. Larger q means more noise budget but weaker security for a given N.

In NINE65, q is never stored as a single integer for production configs (it would overflow u64 for multi-prime setups). Instead, ciphertexts are represented in RNS form as residues modulo each prime separately.

### t --- Plaintext Modulus

The modulus for plaintext values. NINE65 uses t = 65537 (the Fermat prime F4, equal to 2^16 + 1) for all production configs. This gives a plaintext space of 65537 values per polynomial slot.

### eta --- CBD Parameter

The centered binomial distribution parameter. Error polynomials are sampled from CBD(eta), which produces coefficients in the range [-eta, eta]. Larger eta means wider error distribution, which increases noise growth but also increases security margin against certain attacks.

| Config | eta | Error Range |
|--------|-----|-------------|
| secure_128 | 3 | [-3, 3] |
| secure_192 | 4 | [-4, 4] |
| secure_256 | 5 | [-5, 5] |

## NTT-Friendly Primes

For the Number Theoretic Transform to work in the ring Z_q[X]/(X^N + 1), each prime q_i must satisfy:

```
q_i - 1 === 0 (mod 2N)
```

This ensures the existence of a primitive 2N-th root of unity modulo q_i, which is required for the negacyclic NTT. NINE65 verifies this constraint at construction time in `NTTEngine::try_new()` and `NTTEngineFFT::try_new()`.

## HE Standard v1.1 Bounds

The Homomorphic Encryption Standard specifies maximum log2(q) values for each (N, security_level) pair:

| N | 128-bit max log2(q) | 192-bit max log2(q) | 256-bit max log2(q) |
|---|--------------------|--------------------|---------------------|
| 1024 | 27 | 19 | 11 |
| 2048 | 54 | 37 | 29 |
| 4096 | 109 | 75 | 58 |
| 8192 | 218 | 152 | 118 |
| 16384 | 438 | 305 | 237 |

NINE65 validates compliance against these bounds in `HEStandardBounds::is_compliant()`.

## NINE65 Production Configurations

### secure_128

The recommended configuration for most use cases.

| Parameter | Value |
|-----------|-------|
| N | 4096 |
| Primes | 998244353, 985661441, 754974721 |
| Prime sizes | 3 x 30-bit |
| log2(q) | ~90 bits |
| t | 65537 |
| eta | 3 |
| Classical security | 129 bits |
| Quantum security | 86 bits |
| Hybrid security | 129 bits |
| HE Standard compliant | Yes (90 < 109 max for N=4096) |

### secure_128_deep

Extended version for deeper circuits, using N=8192 to maintain security with a larger modulus.

| Parameter | Value |
|-----------|-------|
| N | 8192 |
| Primes | 998244353, 985661441, 754974721, 469762049 |
| Prime sizes | 4 x 30-bit |
| log2(q) | ~120 bits |
| t | 65537 |
| eta | 3 |

### secure_192

High-security configuration for long-term protection.

| Parameter | Value |
|-----------|-------|
| N | 16384 |
| Primes | 998244353, 985661441, 754974721, 469762049, 167772161 |
| Prime sizes | 5 x 28--30-bit |
| log2(q) | ~147 bits |
| t | 65537 |
| eta | 4 |
| Classical security | 374 bits |
| Quantum security | 213 bits |
| Hybrid security | 318 bits |

### secure_256

Maximum security. Significantly slower than lower tiers.

| Parameter | Value |
|-----------|-------|
| N | 16384 |
| Primes | 998244353, 985661441, 754974721, 469762049, 167772161, 595591169 |
| Prime sizes | 6 x 28--30-bit |
| log2(q) | ~177 bits |
| t | 65537 |
| eta | 5 |
| Classical security | 311 bits |
| Quantum security | 177 bits |
| Hybrid security | 264 bits |
| HE Standard compliant | Yes (177 < 237 max for N=16384) |

Note: secure_256 uses 6 primes rather than 7. With 7 primes (log_q ~207), the ternary-secret MITM penalty would reduce hybrid security to 226 bits, which falls below the 230-bit minimum (256 x 90% threshold).

## Construction-Time Verification

`SecureConfig::new_verified()` runs the lattice security estimator at construction time. This means you cannot create a `SecureConfig` with parameters that fail security checks:

```rust
// This runs the estimator and verifies the claimed 128-bit security
let config = SecureConfig::secure_128();
assert!(config.is_production_safe());
```

In release builds without the `allow_insecure` feature flag, the constructor panics if the estimated hybrid security is below 90% of the claimed level. Test configurations (`test_fast_insecure`, `test_medium_insecure`) are gated behind `#[cfg(any(test, debug_assertions, feature = "allow_insecure"))]` and cannot be constructed in release builds.

## Choosing Parameters

- **Default choice**: `SecureConfig::secure_128()`. Provides 128-bit security with good performance.
- **Deeper circuits**: `SecureConfig::secure_128_deep()`. Trades some speed for more multiplicative depth.
- **Long-term security**: `SecureConfig::secure_192()`. Use when data must remain secure for decades.
- **Maximum security**: `SecureConfig::secure_256()`. Use when regulatory requirements demand the highest tier.

## Where to go next

- [BFV Scheme](bfv-scheme) --- how the BFV encryption/decryption/evaluation pipeline works
- [K-Elimination](k-elimination) --- how NINE65 achieves exact division in the RNS representation
- [Security Configs](security-configs) --- deeper dive into the lattice security estimator
