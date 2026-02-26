---
layout: default
title: NTT Engine
parent: How It Works
nav_order: 5
---

# NTT Engine

[Home](index)

The Number Theoretic Transform (NTT) is the polynomial multiplication engine at the heart of NINE65. Without NTT, every polynomial multiplication would be O(N^2). With NTT, it is O(N log N).

## What NTT Does

BFV FHE requires multiplying polynomials in the ring Z_q[X]/(X^N + 1). A naive multiplication of two degree-N polynomials takes O(N^2) coefficient multiplications. For N = 4096, that is 16 million multiplications per polynomial multiply.

The NTT converts polynomials from **coefficient domain** to **evaluation domain** (also called NTT domain or point-value form). In evaluation domain, polynomial multiplication becomes pointwise multiplication --- O(N) operations. The full pipeline is:

```
Coefficient domain        Evaluation domain        Coefficient domain
    a(X)         --NTT-->    A[i]
    b(X)         --NTT-->    B[i]
                             C[i] = A[i] * B[i]    (pointwise, O(N))
                 <--INTT--   c(X)                   (result polynomial)
```

Total cost: two forward NTTs + N pointwise multiplications + one inverse NTT = O(N log N).

## Coefficient Domain vs Evaluation Domain

- **Coefficient domain**: a polynomial a(X) = a_0 + a_1*X + a_2*X^2 + ... + a_{N-1}*X^{N-1}. The natural representation for encoding and decoding.
- **Evaluation domain**: the polynomial evaluated at N specific points (the N-th roots of unity modulo q). The natural representation for multiplication.

In NINE65, `RingPolynomial` values are **always stored in coefficient domain**. The NTT transform is performed internally by the engine when multiplication is requested. This invariant is maintained throughout the codebase.

## Negacyclic Convolution

Because NINE65 works in the ring Z_q[X]/(X^N + 1) (not Z_q[X]/(X^N - 1)), a standard cyclic NTT would give the wrong result. The fix is the **psi-twist**:

1. Before the forward NTT, multiply each coefficient a_i by psi^i, where psi is a primitive 2N-th root of unity
2. Perform a standard NTT using omega = psi^2 (a primitive N-th root of unity)
3. After the inverse NTT, multiply each coefficient c_i by psi^{-i}

This transforms the negacyclic convolution into a cyclic one that the NTT can handle.

## Two Engines

NINE65 provides two NTT implementations, selected at compile time by the `ntt_fft` feature flag (enabled by default).

### NTTEngine (ntt.rs) --- DFT-Based, O(N^2)

The basic engine uses the discrete Fourier transform matrix directly. For each output coefficient k, it computes:

```
A[k] = sum_{j=0}^{N-1} a[j] * omega^{k*j}  (mod q)
```

This is O(N^2) per transform. It is correct and simple, but slow for large N.

Properties:
- Stores: Montgomery context, Barrett context, precomputed omega/psi powers
- Has constant-time variants: `ntt_ct()`, `intt_ct()`, `multiply_ct()` using Barrett constant-time reduction
- Used as the reference implementation for verifying the FFT engine

### NTTEngineFFT (ntt_fft.rs) --- Cooley-Tukey, O(N log N)

The production engine uses the Cooley-Tukey decimation-in-time butterfly algorithm. Expected speedup: 500--2000x over the DFT engine depending on N.

The algorithm proceeds in log2(N) stages. Each stage performs N/2 butterfly operations:

```
For each butterfly (u, v) with twiddle factor w:
    a[u] = a[u] + w * a[v]
    a[v] = a[u] - w * a[v]
```

Key implementation details:

- **All butterfly operations use Montgomery form.** Twiddle factors are precomputed in Montgomery form. The `mont_add` and `mont_sub` helper functions keep values in Montgomery form throughout.
- **Bit-reversal permutation** is applied before the butterfly stages.
- **PersistentMontgomery** integration: the engine holds a `PersistentMontgomery` context and precomputes psi powers in Montgomery form for the twist/untwist steps.
- **N^{-1} scaling** after the inverse NTT uses Montgomery multiplication.

### Feature Flag Selection

```rust
// When ntt_fft is enabled (default):
#[cfg(feature = "ntt_fft")]
use crate::arithmetic::NTTEngineFFT as NTTEngine;

// When ntt_fft is disabled:
#[cfg(not(feature = "ntt_fft"))]
use crate::arithmetic::NTTEngine;
```

Both engines expose the same public API, so the rest of the codebase uses `NTTEngine` transparently. When `ntt_fft` is enabled, `NTTEngineFFT` is re-exported as `NTTEngine` through the prelude.

## NTT-Friendly Primes

For the NTT to work, each ciphertext modulus prime q must satisfy:

```
(q - 1) mod (2 * N) == 0
```

This guarantees the existence of a primitive 2N-th root of unity modulo q. NINE65 verifies this in the constructor and returns `Err(NTTConfigError)` if the constraint is violated.

The standard primes used across all NINE65 configs:

| Prime | Bits | Factorization of q-1 |
|-------|------|---------------------|
| 998244353 | 30 | 2^23 x 7 x 17 |
| 985661441 | 30 | 2^22 x 5 x 47 |
| 754974721 | 30 | 2^24 x 45 |
| 469762049 | 29 | 2^26 x 7 |
| 167772161 | 28 | 2^25 x 5 |
| 595591169 | 30 | NTT-compatible for N=16384 |

## Thread Safety

Both `NTTEngine` and `NTTEngineFFT` are `Send + Sync`. They are immutable after construction --- all methods take `&self`. Multiple threads can safely share an NTT engine reference via `Arc<NTTEngine>` for parallel polynomial operations.

## Montgomery Form in Butterflies

The FFT engine performs all butterfly multiplications in Montgomery form. This avoids the overhead of converting in and out of Montgomery form for each multiplication. The twiddle factors are precomputed once in Montgomery form during construction:

```rust
let twiddles_fwd = Self::compute_twiddles(&mont, omega, n);
```

Inside each butterfly, `montgomery_mul` is called directly on Montgomery-form values, keeping everything in the fast Montgomery domain throughout the entire transform.

## Where to go next

- [Montgomery Arithmetic](montgomery) --- the constant-time modular arithmetic underneath NTT butterflies
- [BFV Scheme](bfv-scheme) --- how NTT polynomial multiplication fits into the BFV pipeline
- [K-Elimination](k-elimination) --- the exact division algorithm that operates after NTT-based multiplication
