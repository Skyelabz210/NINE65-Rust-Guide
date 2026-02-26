---
layout: default
title: BFV Scheme
parent: How It Works
nav_order: 2
---

# BFV Scheme

[Home](index)

The Brakerski/Fan-Vercauteren (BFV) scheme is the FHE construction that NINE65 implements. This page covers how BFV works mathematically, and the two distinct APIs that NINE65 provides for it.

## Mathematical Setting

BFV operates in two algebraic spaces:

- **Plaintext space** Z_t: integers modulo the plaintext modulus t. In NINE65, t = 65537 for all production configs.
- **Ciphertext space** R_q = Z_q[X]/(X^N + 1): polynomials of degree less than N with coefficients modulo q. The quotient by (X^N + 1) makes this a negacyclic ring.

The critical link between these spaces is the **scaling factor**:

```
delta = floor(q / t)
```

Delta lifts a plaintext value from the small modulus t into the large modulus q, creating room for noise to accumulate without corrupting the message.

## Encoding

A plaintext message m (where 0 <= m < t) is encoded into a polynomial by placing the scaled value in the constant coefficient:

```
encode(m) -> polynomial where coeffs[0] = delta * m mod q
```

Decoding recovers the message by reversing the scaling:

```
decode(poly) -> m = round(t * poly.coeffs[0] / q) mod t
```

The rounding absorbs small noise terms. As long as the noise is less than delta/2, the rounding recovers the exact plaintext.

In NINE65, the `BFVEncoder` struct (in `ops/encrypt.rs`) handles encoding and decoding. It computes the rounding via the integer formula `floor((2*t*c + q) / (2*q)) mod t` to avoid any floating-point arithmetic.

## Encryption

Given a public key pk = (pk0, pk1) and a message m:

1. Encode: `plaintext = delta * m` (as a polynomial)
2. Sample blinding factor: `u` from the ternary distribution {-1, 0, 1}
3. Sample errors: `e1, e2` from the centered binomial distribution CBD(eta)
4. Compute:
   - `c0 = pk0 * u + e1 + plaintext`
   - `c1 = pk1 * u + e2`

The ciphertext is the pair `(c0, c1)`.

The `BFVEncryptor` struct provides multiple encryption entry points:

| Method | RNG Source | Use Case |
|--------|-----------|----------|
| `encrypt(m, harvester)` | ShadowHarvester | Testing with deterministic entropy |
| `encrypt_seeded(m, seed)` | Deterministic seed | Reproducible tests |
| `encrypt_with_rng(m, rng)` | Any `FheRng` impl | Generic interface |
| `encrypt_secure(m)` | OS CSPRNG | **Production** |
| `try_encrypt_secure(m)` | OS CSPRNG | Production (fallible) |

For production deployments, always use `encrypt_secure()` or `try_encrypt_secure()`, which draw randomness from the operating system's CSPRNG.

## Decryption

Given secret key s and ciphertext (c0, c1):

1. Compute the noisy plaintext: `m_noisy = c0 + c1 * s`
2. Decode: `m = round(t * m_noisy[0] / q) mod t`

The `BFVDecryptor` uses constant-time polynomial multiplication (`mul_ct`) for the `c1 * s` step to prevent timing side-channels from leaking information about the secret key.

## Homomorphic Addition

Adding two ciphertexts is component-wise addition:

```
ct_add = (ct1.c0 + ct2.c0, ct1.c1 + ct2.c1)
```

Noise grows linearly: the noise in the sum is approximately the sum of the individual noises.

## Homomorphic Multiplication

Multiplication is more involved. It produces a degree-2 ciphertext via the **tensor product**:

```
d0 = ct1.c0 * ct2.c0
d1 = ct1.c0 * ct2.c1 + ct1.c1 * ct2.c0
d2 = ct1.c1 * ct2.c1
```

This yields three components (d0, d1, d2) instead of the usual two. The result is at the delta-squared level, so it must be **rescaled** by dividing out one factor of delta. This is where K-Elimination provides exact division.

After rescaling, **relinearization** converts the degree-2 ciphertext back to degree 1 using the evaluation key. This is necessary to keep ciphertext size constant.

Noise grows multiplicatively: each multiplication roughly squares the noise. This is why deep circuits eventually need bootstrap to refresh the noise budget.

## The Two Encryption APIs

NINE65 provides two distinct APIs for different use cases.

### Basic BFV (ops/encrypt.rs)

The basic API operates on a single ciphertext modulus q. Types:

| Type | Purpose |
|------|---------|
| `BFVEncoder` | Message-to-polynomial encoding with delta scaling |
| `BFVEncryptor` | Encryption using public key + NTT engine |
| `BFVDecryptor` | Decryption using secret key + NTT engine |
| `Ciphertext` | A pair (c0, c1) of `RingPolynomial` values |

This API is suitable for learning how BFV works, for simple single-modulus configurations, and as the building block used internally by the bootstrap system.

### Dual-RNS FHE (ops/rns_fhe.rs)

The production API uses the Residue Number System (RNS) with K-Elimination. Types:

| Type | Purpose |
|------|---------|
| `RNSFHEContext` | Central context holding NTT engines, parameters, and RNS configuration |
| `DualRNSCiphertext` | Ciphertext with main + anchor RNS limbs |
| `DualRNSSecretKey` | Secret key in dual-RNS form (ZeroizeOnDrop) |
| `DualRNSPublicKey` | Public key in dual-RNS form |
| `DualRNSEvalKey` | Evaluation key for public relinearization |
| `DualRNSFullKeySet` | Complete key set including eval key for multi-party FHE |

The Dual-RNS API represents every polynomial as a tuple of residues modulo the main RNS primes **and** a separate tuple of residues modulo the anchor primes. This dual-track representation enables exact K-Elimination division after tensor product, even when the intermediate values exceed any single modulus.

The `MulRoute` enum selects between two multiplication strategies automatically:

- **BajardSingle**: Single-RNS per-limb rescaling. Faster but approximate. Used when delta-squared fits within the RNS product.
- **KElimDual**: Dual-RNS K-Elimination rescaling. Exact. Required when delta-squared exceeds the main RNS product, which is the common case for production parameters.

### Evaluator Variants (ops/homomorphic.rs)

| Type | Description |
|------|-------------|
| `BFVEvaluator` | Unchecked homomorphic operations --- fastest path, no noise tracking |
| `TrackedEvaluator` | Wraps `BFVEvaluator` with a `NoiseBudget` tracker; `try_mul()` returns `Result` when noise is exhausted |

The `TrackedEvaluator` is recommended for circuits where you need to detect noise exhaustion before it silently corrupts the plaintext. It tracks a `NoiseBudget` that decreases with each operation, and returns `Err(NoiseExhausted)` when the budget reaches zero.

## Thread Safety

`Ciphertext` and `DualRNSCiphertext` are both `Send + Sync`. They can be safely sent between threads and shared via `Arc` for concurrent read access. Clone before modifying from different threads.

## Where to go next

- [BFV Parameters](bfv-params) --- how the parameter values (N, q, t, eta) were chosen and what they mean for security
- [K-Elimination](k-elimination) --- the exact division algorithm that makes Dual-RNS multiplication work
- [NTT Engine](ntt) --- the polynomial multiplication engine underneath BFV
