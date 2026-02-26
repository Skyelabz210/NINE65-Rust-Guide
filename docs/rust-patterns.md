---
layout: default
title: Rust Patterns
parent: Foundation
nav_order: 5
---

# Rust Patterns

[Home](index)

Rust language features as they appear in the NINE65 codebase. Each pattern is shown with a real example from the source.

---

## Nine65Result\<T\> and the ? operator

The entire codebase uses a unified error type defined in `crates/nine65/src/errors.rs`:

```rust
pub type Nine65Result<T> = Result<T, Nine65Error>;
```

Functions that can fail return `Nine65Result<T>`. The `?` operator propagates errors up the call stack without manual matching:

```rust
pub fn try_new(config: &FHEConfig) -> Nine65Result<Self> {
    let dual_rns = DualRNSContext::for_fhe(&config.primes, config.n)?;
    // ... if for_fhe() returns Err, try_new() returns that Err immediately
    Ok(Self { dual_rns, /* ... */ })
}
```

Convenience wrappers like `RNSFHEContext::new()` call `try_new()` and panic on error, so callers who want to handle errors gracefully use `try_new()` and those who want fail-fast behavior use `new()`.

---

## Nine65Error and thiserror

The error enum uses the `thiserror::Error` derive macro for automatic `Display` and `Error` implementations. Each variant maps to a formal precondition from the Coq/Lean4 proofs:

```rust
use thiserror::Error;

#[derive(Debug, Clone, Error)]
pub enum Nine65Error {
    #[error("coprimality violation: gcd({m}, {a}) = {gcd} != 1")]
    NotCoprime { m: u64, a: u64, gcd: u64 },

    #[error("range overflow: X={x} >= M*A={bound}")]
    RangeOverflow { x: u128, bound: u128 },

    #[error("noise budget exhausted: needed {required_mb} millibits, had {available_mb}")]
    NoiseBudgetExhausted { required_mb: i64, available_mb: i64 },

    // ... 20+ variants covering K-Elimination, GSO-FHE, encoding,
    //     bootstrap, entropy, serialization, regime mismatches
}
```

Each variant has structured fields (not stringly-typed) so callers can match on specific error data:

```rust
match err {
    Nine65Error::NoiseBudgetExhausted { required_mb, available_mb } => {
        // decide whether to bootstrap or abort
    }
    _ => return Err(err),
}
```

The error also provides classification methods:

```rust
impl Nine65Error {
    pub fn is_recoverable(&self) -> bool { /* ... */ }
    pub fn is_batching_error(&self) -> bool { /* ... */ }
    pub fn category(&self) -> &'static str { /* ... */ }
}
```

---

## Feature gating with cfg

Test-only and insecure configurations are blocked at compile time in release builds using nested `cfg` attributes:

```rust
#[cfg(any(test, debug_assertions, feature = "allow_insecure"))]
pub fn test_fast_insecure() -> Self {
    Self::new_verified(1024, vec![998244353], 65537, 2, 40, "test_fast_insecure")
}
```

This function only exists when:
- Running tests (`#[cfg(test)]`)
- In debug mode (`#[cfg(debug_assertions)]`)
- The `allow_insecure` feature is explicitly enabled

In a release build without `allow_insecure`, calling `SecureConfig::test_fast_insecure()` produces a compile error --- the function does not exist. This is a compile-time security gate, not a runtime check.

The same pattern gates diagnostic output and debug helpers:

```rust
#[cfg(any(test, debug_assertions))]
pub fn decrypt_dual_with_diagnostics(
    &self,
    ct: &DualRNSCiphertext,
    sk: &DualRNSSecretKey,
) -> Nine65Result<(u64, i128)> { /* ... */ }
```

---

## Zeroize and ZeroizeOnDrop

Secret keys must be erased from memory when no longer needed. The `zeroize` crate provides this via derive macros:

```rust
use zeroize::{Zeroize, ZeroizeOnDrop};

#[derive(Clone, Zeroize, ZeroizeOnDrop)]
pub struct SecretKey {
    pub s: RingPolynomial,
}
```

`Zeroize` adds a `.zeroize()` method that overwrites the memory with zeros using volatile writes the compiler cannot optimize away. `ZeroizeOnDrop` automatically calls `.zeroize()` when the value goes out of scope.

The same pattern applies to `RNSSecretKey` and `DualRNSSecretKey`:

```rust
#[derive(Clone, Zeroize, ZeroizeOnDrop)]
pub struct RNSSecretKey {
    pub s: RNSPolynomial,
}
```

The underlying `RingPolynomial` implements `Zeroize` on its `Vec<u64>` coefficients, so the entire chain from key struct down to raw coefficient memory is covered.

---

## Workspace-level enforcement

Two crate-level attributes enforce invariants across the entire nine65 crate:

```rust
// crates/nine65/src/lib.rs
#![forbid(unsafe_code)]
```

`forbid` is stronger than `deny` --- it cannot be overridden by inner `#[allow(...)]` attributes. Any `unsafe` block anywhere in the `nine65` crate is a hard compile error.

For float prohibition, `clockwork-core` and `fhe-service` enforce it at the compiler level:

```rust
// crates/clockwork-core/src/lib.rs
#![deny(unsafe_code)]
#![deny(clippy::float_arithmetic)]
#![deny(clippy::float_cmp)]
```

The `nine65` core crate achieves its zero-float guarantee architecturally: every computation path uses integer arithmetic, and the circuit compiler (the one module that uses `f64` for offline static noise analysis) is isolated from the cryptographic runtime.

Other crates in the workspace enforce the same discipline:

```rust
// crates/mana/src/lib.rs
#![forbid(unsafe_code)]
#![deny(missing_docs)]

// crates/nexgen_rational/src/lib.rs
#![forbid(unsafe_code)]
```

---

## pub(crate) visibility

Types and functions that need to be shared across modules within a crate but should not be part of the public API use `pub(crate)`:

```rust
// crates/nine65/src/arithmetic/rns.rs
pub(crate) struct U256 {
    pub(crate) lo: u128,
    pub(crate) hi: u128,
}
```

`U256` is an internal 256-bit unsigned integer used for intermediate CRT reconstruction. Other modules within `nine65` (like `rns_fhe.rs`) can use it, but downstream users cannot. The re-export in `arithmetic/mod.rs` makes this explicit:

```rust
pub(crate) use rns::{compute_delta_rns_overflow_safe, U256};
```

Similarly, internal bootstrap helpers are scoped to the crate:

```rust
// crates/nine65/src/ops/bootstrap.rs
pub(crate) fn modswitch_to_t(&self, ct: &DualRNSCiphertext) -> Nine65Result<(Vec<u64>, Vec<u64>)> {
    // ...
}
```

---

## Trait objects: the FheRng trait

The entropy system defines a trait that abstracts over randomness sources:

```rust
// crates/nine65/src/entropy/rng_trait.rs
pub trait FheRng {
    fn next_u64(&mut self) -> u64;
    fn uniform(&mut self, bound: u64) -> u64;
    fn ternary(&mut self) -> i64;
    fn cbd(&mut self, eta: usize) -> i64;
}
```

Both `ShadowHarvester` (deterministic, fast, test-only) and `SecureRng` (OS CSPRNG, production) implement this trait. Functions that need randomness are generic over it:

```rust
pub fn generate_keys_dual_with_rng<R: FheRng>(&self, rng: &mut R) -> DualRNSKeySet { /* ... */ }
pub fn encrypt_dual_with_rng<R: FheRng>(&self, m: u64, pk: &DualRNSPublicKey, rng: &mut R)
    -> DualRNSCiphertext { /* ... */ }
```

This means production code uses `SecureRng` and tests use `ShadowHarvester` with no code duplication.

---

## Arc\<Vec\<u64\>\> for shared primes

The MANA stream accelerator stores prime moduli in an `Arc` to avoid cloning on every operation:

```rust
// crates/mana/src/stream.rs
pub struct ManaStream {
    pub lanes: Vec<Lane>,
    pub primes: Arc<Vec<u64>>,
    pub n: usize,
    pub product_cache: u128,
}
```

When a `ManaStream` is cloned (which happens frequently in parallel pipelines), the `Arc<Vec<u64>>` is reference-counted --- only the pointer is copied, not the prime vector. This matters because streams are the unit of parallelism in MANA and get distributed across threads.

---

## Lifetimes: BFVEncryptor\<'a\>

The single-modulus BFV encryptor borrows its dependencies rather than owning them:

```rust
// crates/nine65/src/ops/encrypt.rs
pub struct BFVEncryptor<'a> {
    pub pk: &'a PublicKey,
    pub encoder: &'a BFVEncoder,
    pub ntt: &'a NTTEngine,
    pub eta: usize,
}
```

The lifetime `'a` ties the encryptor to the lifetime of its public key, encoder, and NTT engine. The encryptor cannot outlive any of them. This avoids cloning large structures (the NTT engine alone contains precomputed twiddle factor tables for the full polynomial ring).

Usage:

```rust
let ntt = NTTEngine::new(config.q, config.n);
let keys = KeySet::generate(&config, &ntt, &mut rng);
let encoder = BFVEncoder::new(&config);
let encryptor = BFVEncryptor::new(&keys.public_key, &encoder, &ntt, config.eta);
// encryptor borrows ntt, keys.public_key, and encoder
// all must remain alive while encryptor is in use
```

Note: the DualRNS path (`RNSFHEContext`) uses owned data instead of lifetimes, since the context is typically long-lived and owns everything it needs.

---

## The prelude pattern

`crates/nine65/src/lib.rs` defines a `prelude` module that re-exports the most commonly used types:

```rust
pub mod prelude {
    pub use crate::arithmetic::{
        BarrettContext, MontgomeryContext, PersistentMontgomery, RNSContext, RNSPolynomial,
        MQReLU, IntegerSoftmax, PadeEngine, CyclotomicRing, CyclotomicPolynomial,
        // ... ~20 more types
    };

    pub use crate::entropy::ShadowHarvester;
    pub use crate::entropy::SecureRng;

    pub use crate::params::{FHEConfig, SecureConfig};
    pub use crate::keys::{EvaluationKey, KeySet, PublicKey, SecretKey};

    pub use crate::ops::{BFVDecryptor, BFVEncoder, BFVEncryptor, BFVEvaluator, Ciphertext};
    pub use crate::ops::rns_fhe::{
        RNSFHEContext, DualRNSCiphertext, DualRNSKeySet, DualRNSFullKeySet,
        DualRNSSecretKey, DualRNSPublicKey, DualRNSEvalKey, AutoCiphertext, AutoKeys,
    };

    pub use crate::noise::budget::{NoiseBudget, NoiseOpType};
    pub use crate::errors::{Nine65Error, Nine65Result};

    // ... more re-exports
}
```

In application code, a single `use nine65::prelude::*;` brings in everything needed for typical FHE operations. The prelude also conditionally exports types behind feature flags:

```rust
#[cfg(any(test, feature = "deterministic_rng"))]
pub use crate::entropy::DeterministicRng;

#[cfg(feature = "shadow-entropy")]
pub use crate::entropy::WassanNoiseField;
```

---

## Where to go next

- [Feature Flags](feature-flags) --- what each flag enables and which types become available
- [Architecture](architecture) --- how the modules described here fit together into the full FHE pipeline
- [Glossary](glossary) --- definitions for terms like RNS, NTT, K-Elimination, and others used throughout the code
