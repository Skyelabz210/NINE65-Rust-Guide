---
layout: default
title: Architecture
parent: How It Works
nav_order: 1
---

# Architecture


## What NINE65 Is

NINE65 v7 is an unlimited-depth Fully Homomorphic Encryption (FHE) system. It lets you perform computations on encrypted data without decrypting it first. The "unlimited-depth" part means there's no limit on how many operations you can chain together --- the system automatically refreshes ciphertexts via bootstrap when noise gets too high.

## Core Rule: Zero Floats

Every calculation in the entire system uses integer or exact rational arithmetic. No `f32`, no `f64`, anywhere. This is enforced at the compiler level. The reason: floating-point arithmetic introduces rounding errors that compound across operations. In FHE, where you're doing thousands of multiplications on encrypted data, those errors would destroy your results.

## How FHE Works (Simplified)

1. **Encrypt** a plaintext value (e.g., the number 42) into a ciphertext. The ciphertext is a polynomial with added "noise."
2. **Operate** on ciphertexts --- add them, multiply them. Each operation increases the noise.
3. **Decrypt** the result. As long as noise hasn't exceeded the budget, you get the exact answer.

The problem: after a few multiplications, noise exceeds the budget and decryption gives garbage.

The solution: **bootstrap** --- a procedure that takes a noisy ciphertext and produces a fresh one with the same plaintext but reset noise. NINE65 does this automatically.

## The Three Layers

```
Application Layer
    |
    v
RNSFHEContext (BFV operations: encrypt, add, mul, decrypt)
    |
    v
DualRNS Arithmetic (polynomials stored as RNS residues + K-Elimination anchors)
    |
    v
Montgomery / NTT (constant-time modular arithmetic, number-theoretic transforms)
```

### RNS (Residue Number System)

Instead of storing one big number, RNS stores it as residues modulo several small primes. This makes addition and multiplication embarrassingly parallel --- each prime channel is independent.

Example: to represent 42 with primes [5, 7, 11]:
- 42 mod 5 = 2
- 42 mod 7 = 0
- 42 mod 11 = 9
- Stored as (2, 0, 9)

Add or multiply just operates on each channel independently. CRT (Chinese Remainder Theorem) reconstructs the original value when needed.

### K-Elimination

The anchor system for exact division and overflow detection. Uses a separate set of coprime anchor primes to track values that would otherwise require full CRT reconstruction. The "K" value captures the quotient information lost during modular arithmetic.

### Montgomery Form

All modular arithmetic uses Montgomery reduction for constant-time operations. This means no branching on secret data --- critical for security against timing side-channel attacks.

### NTT (Number Theoretic Transform)

Polynomial multiplication via the NTT (integer equivalent of FFT). Converts polynomials from coefficient domain to evaluation domain where multiplication is pointwise. Uses Cooley-Tukey decimation-in-time with Montgomery form throughout.

## Key Data Types

| Type | What it holds | Where |
|------|-------------|-------|
| `DualRNSCiphertext` | Encrypted value (c0, c1 polynomial pair in RNS) | `ops/rns_fhe.rs` |
| `RNSFHEContext` | FHE operations + config | `ops/rns_fhe.rs` |
| `DualRNSEvalKey` | Relinearization key for multiplication | `ops/rns_fhe.rs` |
| `BootstrapKey` (BSK) | Key for homomorphic accumulator | `keys/bootstrap.rs` |
| `KeySwitchKey` (KSK) | Key for switching between key spaces | `keys/bootstrap.rs` |
| `NoiseBudget` | Tracks remaining noise headroom in millibits | `noise/budget.rs` |
| `FHEConfig` | Ring dimension, primes, plaintext modulus, security level | `params/mod.rs` |
| `SecureConfig` | Validated production configurations | `params/secure_configs.rs` |
