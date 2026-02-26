---
layout: default
title: Montgomery Arithmetic
parent: How It Works
nav_order: 6
---

# Montgomery Arithmetic

[Home](index)

Montgomery arithmetic is how NINE65 performs modular multiplication without division. Every modular multiplication in the NTT, in key generation, and in homomorphic operations goes through Montgomery reduction. This page covers the three complementary modular arithmetic systems.

## Why Montgomery Form

Standard modular multiplication computes `a * b mod q` using division (or its equivalent). Division is expensive and, worse, has data-dependent timing on most CPUs. Montgomery form avoids both problems.

The idea: represent a value `a` as `a_mont = a * R mod q` where R = 2^64 (the machine word size). Multiplication of two Montgomery-form values gives:

```
a_mont * b_mont = (a * R) * (b * R) = a * b * R^2
```

Montgomery reduction then computes `(a * b * R^2) * R^{-1} mod q = a * b * R mod q`, which is the product in Montgomery form. The reduction uses only multiplication and bit shifts --- no division by q.

## MontgomeryContext (montgomery.rs)

The basic Montgomery context for a modulus q. Every NTT engine and every Barrett context internally holds a `MontgomeryContext`.

### Precomputed Values

| Field | Value | Purpose |
|-------|-------|---------|
| `q` | The modulus | |
| `r` | 2^64 | Implicit in the representation |
| `r2` | R^2 mod q | Fast conversion to Montgomery form |
| `q_inv_neg` | -q^{-1} mod 2^64 | Core REDC constant |

`q_inv_neg` is computed via Newton's method (6 iterations, converges for any odd q).

### Constant-Time Operations

Every operation in `MontgomeryContext` is constant-time:

- **`montgomery_reduce`**: The REDC algorithm. The final conditional subtraction uses bit manipulation (`overflowing_sub` + mask), not a branch.
- **`montgomery_add`**: Conditional subtraction via `overflowing_sub` + mask.
- **`montgomery_sub`**: Conditional addition via wrapping with mask.
- **`montgomery_neg`**: Zero-check via mask, not branch.
- **`montgomery_pow`**: Uses the **Montgomery ladder** algorithm where the execution path is independent of the exponent bits. Both branches (multiply and square) execute every iteration; a constant-time swap selects which result goes where.
- **`montgomery_pow_vartime`**: A faster variant for public exponents only (like NTT root computation). Uses standard square-and-multiply with data-dependent branches.

### Coq Proof

Montgomery correctness is verified in `proofs/coq/Montgomery.v`.

## PersistentMontgomery (persistent_montgomery.rs)

The traditional approach to Montgomery arithmetic converts values in and out of Montgomery form around each operation:

```
to_montgomery(a) -> compute -> compute -> compute -> from_montgomery(result)
```

Persistent Montgomery eliminates this overhead entirely. Values **stay in Montgomery form** permanently. Conversion only happens at true system boundaries (I/O, interop with external systems).

```
Traditional:   enter -> op -> exit -> enter -> op -> exit -> enter -> op -> exit
Persistent:    enter -> op -> op -> op -> op -> op -> op -> exit
```

### The PersistentMontgomery Struct

```rust
pub struct PersistentMontgomery {
    pub m: u64,           // Modulus
    pub r_squared: u64,   // R^2 mod m (for lazy entry)
    pub m_prime: u64,     // m' such that m * m' = -1 (mod R)
    pub r_log: u32,       // log2(R) = 64
}
```

### Persistent Operations

All operations take Montgomery-form inputs and produce Montgomery-form outputs:

| Method | Operation | Notes |
|--------|-----------|-------|
| `mul(x, y)` | x * y in Montgomery form | Core REDC |
| `add(x, y)` | x + y in Montgomery form | Simple conditional subtract |
| `sub(x, y)` | x - y in Montgomery form | Simple conditional add |
| `neg(x)` | -x in Montgomery form | |
| `square(x)` | x^2 in Montgomery form | Slightly faster than mul |
| `pow(base, exp)` | base^exp in Montgomery form | Square-and-multiply |
| `inverse(x)` | x^{-1} via Fermat's little theorem | x^{m-2} mod m |

### Integration with NTT

`NTTEngineFFT` holds a `PersistentMontgomery` instance and precomputes all twiddle factors in Montgomery form. The butterfly operations (`ntt_inplace`, `intt_inplace`) perform all multiplications via `montgomery_mul` on values that are already in Montgomery form. This means the entire forward and inverse NTT runs in Montgomery space with zero conversion overhead.

## BarrettContext (barrett.rs)

Barrett reduction is the complement to Montgomery for cases where values are not staying in a persistent domain. It computes `a mod q` for `a < q^2` using a precomputed reciprocal:

```
q_hat = floor(a * mu / 2^128)    where mu = floor(2^128 / q)
r = a - q_hat * q
```

### When to Use Barrett vs Montgomery

| Situation | Use |
|-----------|-----|
| Repeated multiplications (NTT, polynomial ops) | Montgomery (amortize conversion cost) |
| One-off reduction of an accumulator | Barrett |
| Secret data processing | Constant-time Barrett (`reduce_ct`) or Montgomery |
| NTT engine internals | Both (Montgomery for butterflies, Barrett for CT variants) |

### Constant-Time Barrett

`reduce_ct()` uses bit-mask conditional subtraction instead of branching:

```rust
let mask1 = ((result >= self.q) as u64).wrapping_neg();
result = result.wrapping_sub(self.q & mask1);
```

This prevents timing side-channels when reducing values derived from secret data.

### HybridModContext

`HybridModContext` combines Barrett and Montgomery in a single context, allowing the caller to choose the best reduction strategy per-operation. It is used internally by modules that need both reduction styles.

## Security Properties

The constant-time discipline is critical for FHE security:

1. **Secret key operations** (decryption, key generation) use constant-time multiplication to prevent timing attacks that could recover the secret key.
2. **NTT butterflies** on secret polynomials use `ntt_ct()` / `intt_ct()` which route through Barrett constant-time reduction.
3. **Montgomery ladder exponentiation** ensures that power analysis cannot recover exponents that may be secret.

All of these protections operate at the arithmetic level, complementing the higher-level protections in the Three-Lock bootstrap and GRO timing gates.

## Where to go next

- [NTT Engine](ntt) --- how Montgomery arithmetic powers the polynomial multiplication engine
- [Three-Lock Bootstrap](three-lock) --- the defense-in-depth system that protects secrets during re-encryption
- [BFV Scheme](bfv-scheme) --- how modular arithmetic fits into the full FHE pipeline
