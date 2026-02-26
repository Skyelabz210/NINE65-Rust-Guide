---
layout: default
title: Batch and Galois
parent: Using the System
nav_order: 4
---

# Batch Encoding and Galois Rotations

[Home](index)

Pack multiple values into a single ciphertext and rotate slots without decryption.

---

## BatchEncoder

Defined in `ops/batch.rs`. Packs up to N values into one ciphertext by placing each value as a scaled coefficient of a polynomial.

### Construction

```rust
use nine65::ops::batch::BatchEncoder;

let encoder = BatchEncoder::new(&config)?;

// Maximum values per ciphertext
let cap = encoder.capacity(); // == config.n (e.g. 1024 or 4096)
```

### Encoding

Each value is scaled by Delta = q/t and placed into a polynomial coefficient:

```
p(X) = Delta * v[0] + Delta * v[1] * X + Delta * v[2] * X^2 + ...
```

```rust
let values = vec![10u64, 20, 30, 40];
let poly = encoder.encode(&values)?;

// With explicit zero-padding documentation
let poly = encoder.encode_padded(&values)?;
```

Constraints:
- `values.len()` must be `<= encoder.capacity()`
- Each value must be `< config.t`

Errors:
- `Nine65Error::TooManySlotValues` if too many values
- `Nine65Error::MessageOutOfBounds` if any value >= t

### Decoding

Recovers values by unscaling each coefficient: `v[i] = round(coeff[i] * t / q) mod t`.

```rust
// Decode first 4 values
let decoded = encoder.decode(&result_poly, 4);

// Decode all N coefficients
let all = encoder.decode_all(&result_poly);
```

---

## Coefficient Batching vs SIMD Slot Batching

NINE65 currently implements **coefficient batching**. The distinction matters for multiplication.

### Coefficient Batching (current)

- Packs values into polynomial coefficients
- **Add/Sub**: coefficient-wise (SIMD-like for add/sub)
- **Mul**: polynomial product (convolution, NOT element-wise)
- Works with any plaintext modulus t

After encrypting `poly_a` and `poly_b` and computing `ct_a + ct_b`, decoding gives element-wise sums. But `ct_a * ct_b` gives the polynomial product, which mixes coefficients via convolution.

### SIMD Slot Batching (future)

- Requires `t = 1 (mod 2N)` for the ring to factor into independent slots
- **Add/Sub**: slot-wise
- **Mul**: slot-wise (true SIMD)
- Planned for v0.3

Check if a config supports SIMD:

```rust
let has_simd = BatchEncoder::supports_simd_slots(&config);
// true only when (config.t - 1) % (2 * config.n) == 0
```

### Practical Impact

| Operation | Coefficient Batching | SIMD Slot Batching |
|-----------|---------------------|-------------------|
| Encrypt N values | 1 ciphertext | 1 ciphertext |
| Homomorphic add | Element-wise | Element-wise |
| Homomorphic mul | Polynomial product (convolution) | Element-wise |
| Required t | Any | `t = 1 (mod 2N)` |

For workloads that only need homomorphic addition on batched data, coefficient batching is fully sufficient.

---

## Galois Automorphisms

Defined in `ops/galois.rs`. Galois automorphisms enable rotating the "slots" of a ciphertext without decryption.

### Mathematical Foundation

For the ring `R_q = Z_q[X] / (X^N + 1)`, the Galois automorphism sigma_k maps:

```
sigma_k: X -> X^k    (where k is odd and coprime to 2N)
```

### Rotation Exponents

- **Left rotation by r**: use exponent `5^r mod 2N` (5 generates Z*_{2N})
- **Right rotation by r**: use exponent `5^(-r) mod 2N`
- **Conjugation**: use exponent `2N - 1`

### GaloisEngine

Computes automorphisms on polynomial coefficients.

```rust
use nine65::ops::galois::GaloisEngine;

let galois = GaloisEngine::new(config.n, config.q);
```

The engine handles the index mapping: applying sigma_k permutes coefficients based on `k` and the ring structure `X^N + 1`, where powers exceeding N get negated.

### GaloisKey

Each rotation requires a dedicated key-switching key. Generated from the secret key for a specific automorphism exponent.

```rust
use nine65::ops::galois::{GaloisKey, GaloisKeySet};

pub struct GaloisKey {
    pub exponent: usize,
    pub key_switch_a: Vec<RingPolynomial>,
    pub key_switch_b: Vec<RingPolynomial>,
}
```

Key fields:
- `exponent`: the automorphism exponent k (must be odd)
- `key_switch_a`, `key_switch_b`: gadget decomposition components where `b = -a*s + e + s(X^k) * P/q`

### GaloisKeySet

A collection of Galois keys indexed by exponent.

```rust
let mut key_set = GaloisKeySet::new();
key_set.add_key(galois_key_for_rotation_1);
key_set.add_key(galois_key_for_rotation_2);

// Query
if key_set.has_exponent(5) {
    let gk = key_set.get(5).unwrap();
}

// List all supported exponents
let exponents = key_set.supported_exponents();
```

### GaloisEvaluator

Combines a `GaloisEngine`, an `NTTEngine`, and a `GaloisKeySet` to perform rotations on ciphertexts.

```rust
use nine65::ops::galois::GaloisEvaluator;

let evaluator = GaloisEvaluator::new(&galois, &ntt, &key_set);
```

### Validation

Both `GaloisKey` and `GaloisKeySet` support validation after deserialization:

```rust
// Validate a single key
galois_key.validate(expected_n, expected_q)?;

// Validate entire set
key_set.validate(expected_n, expected_q)?;
```

Validation checks:
- Exponent is odd (required for valid automorphism in `X^N + 1`)
- `key_switch_a` and `key_switch_b` have matching lengths
- All polynomials have degree N
- All coefficients are `< q`

### Serialization

With the `serde` feature enabled:

```rust
// Serialize
let json = galois_key.to_json()?;
let bytes = galois_key.to_bytes()?;

// Deserialize with validation (recommended for production)
let key = GaloisKey::from_json_validated(&json, n, q)?;
let key = GaloisKey::from_bytes_validated(&bytes, n, q)?;

// GaloisKeySet supports the same patterns
let set = GaloisKeySet::from_json_validated(&json, n, q)?;
let set = GaloisKeySet::from_bytes_validated(&bytes, n, q)?;
```

Unvalidated deserialization is gated behind the `allow_insecure` feature and is deprecated.

---

## Where to go next

- [Cookbook](cookbook) -- batch encoding recipe in context
- [Neural Operations](neural-ops) -- neural layers that operate on batched encrypted data
- [Key Management](key-management) -- GaloisKey generation and lifecycle
