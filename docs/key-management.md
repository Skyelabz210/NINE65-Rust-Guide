---
layout: default
title: Key Management
parent: Using the System
nav_order: 2
---

# Key Management

[Home](index)

Every key type in NINE65, how to generate them, and how to keep them safe.

---

## Key Types at a Glance

| Type | Module | Purpose |
|------|--------|---------|
| `SecretKey` | `keys/mod.rs` | Ternary polynomial `s` -- decryption capability |
| `PublicKey` | `keys/mod.rs` | Pair `(pk0, pk1)` -- encryption capability |
| `EvaluationKey` / `DualRNSEvalKey` | `ops/rns_fhe.rs` | Relinearization after ct*ct multiplication |
| `BootstrapKey` | `keys/bootstrap.rs` | Working sk encrypted under bootstrap params |
| `KeySwitchKey` | `keys/bootstrap.rs` | Converts ciphertext from boot key to work key |
| `GaloisKey` | `ops/galois.rs` | Key switching after automorphism (slot rotation) |
| `GaloisKeySet` | `ops/galois.rs` | Collection of Galois keys for supported rotations |

---

## SecretKey

A ternary polynomial `s` with coefficients in `{-1, 0, 1}`. This is the only key that must be kept secret. All other keys are derived from it but can be shared.

```rust
use nine65::keys::SecretKey;
use nine65::entropy::ShadowHarvester;

// TESTING: deterministic (ShadowHarvester seed)
let mut rng = ShadowHarvester::with_seed(42);
let sk = SecretKey::generate(&config, &mut rng);

// PRODUCTION: OS CSPRNG (getrandom)
let sk = SecretKey::generate_secure(&config);

// Fallible variant (returns Nine65Result)
let sk = SecretKey::try_generate_secure(&config)?;
```

`SecretKey` implements `Zeroize` and `ZeroizeOnDrop`. When the key is dropped, its polynomial coefficients are overwritten with zeros using volatile writes that the compiler cannot optimize away.

---

## PublicKey

A pair `(pk0, pk1)` where `pk0 = -a*s + e` and `pk1 = a`. The secret key `s` is protected during generation by constant-time polynomial multiplication (`mul_ct`).

```rust
use nine65::keys::PublicKey;

// TESTING
let pk = PublicKey::generate(&sk, &config, &ntt, &mut rng);

// PRODUCTION
let pk = PublicKey::generate_secure(&sk, &config, &ntt);
```

---

## DualRNS Key Types

For RNS-native FHE (the recommended path), keys exist in dual-track form with both main and anchor residues.

### DualRNSSecretKey

The secret polynomial stored as both main RNS limbs and anchor RNS limbs. Implements `Zeroize + ZeroizeOnDrop`.

### DualRNSPublicKey

Pair `(pk0, pk1)` in dual-RNS form. Supports `serde` serialization when the `serde` feature is enabled.

### DualRNSEvalKey

Relinearization key for public-mode multiplication. Contains gadget decomposition components:

```
rlk[i] = (rlk0_i, rlk1_i)
where rlk0_i = -a_i*s - e_i + power_i * s^2, rlk1_i = a_i
```

Fields:
- `rlk`: Vector of `(DualRNSPoly, DualRNSPoly)` pairs
- `decomp_base`: Decomposition base (default `2^16`, smaller = less noise but larger key)
- `num_digits`: Number of decomposition digits

### Key Set Types

```rust
// Symmetric mode (single-party -- testing/benchmarks)
pub struct DualRNSKeySet {
    pub secret_key: DualRNSSecretKey,
    pub public_key: DualRNSPublicKey,
}

// Public mode (multi-party FHE -- production)
pub struct DualRNSFullKeySet {
    pub secret_key: DualRNSSecretKey,
    pub public_key: DualRNSPublicKey,
    pub eval_key: DualRNSEvalKey,
}
```

### Generation

```rust
let ctx = RNSFHEContext::new(&config);

// Symmetric (no eval key -- cannot do public multiplication)
let keys = ctx.generate_keys_dual(&mut rng);
let keys = ctx.generate_keys_dual_secure();

// Full (with eval key -- supports public multiplication)
let keys = ctx.generate_keys_dual_full(&mut rng);
let keys = ctx.generate_keys_dual_full_secure();

// Deep circuits (smaller decomp base for less relin noise)
let keys = ctx.generate_keys_dual_full_public_deep(&mut rng);
let keys = ctx.generate_keys_dual_full_public_deep_secure();

// Custom decomposition base
let keys = ctx.generate_keys_dual_full_with_base(&mut rng, 256); // base=256
let keys = ctx.generate_keys_dual_full_with_base_secure(256);
```

---

## BootstrapKey and KeySwitchKey

Defined in `keys/bootstrap.rs`. Required for the Clockwork Bootstrap system.

### BootstrapKey

The working secret key encrypted under bootstrap parameters. Since `s` has ternary coefficients, encrypting it adds minimal noise.

```rust
pub struct BootstrapKey {
    pub enc_s: DualRNSCiphertext,       // Encrypted working sk
    pub eval_key: DualRNSEvalKey,       // Eval key for bootstrap circuit
    pub public_key: DualRNSPublicKey,   // Bootstrap public key
    pub t_work: u64,                    // Working plaintext modulus
    pub q_min: u128,                    // Minimum modulus (bootstrap trigger)
}
```

### KeySwitchKey

Converts a ciphertext encrypted under the bootstrap key back to the working key:

```rust
pub struct KeySwitchKey {
    pub ksk: Vec<(DualRNSPoly, DualRNSPoly)>,  // Key-switch components
    pub decomp_base: u64,
    pub num_digits: usize,
}
```

Each component: `ksk[l] = (b_l, a_l)` where `b_l = -a_l*s_work + e_l + s_boot * base^l`.

### BootstrapKeySet

Bundles everything needed for bootstrap:

```rust
pub struct BootstrapKeySet {
    pub bsk: BootstrapKey,
    pub ksk: KeySwitchKey,
    pub boot_sk: DualRNSSecretKey,  // For testing only; discard in production
}
```

---

## BOOTSTRAP_PRIMES

Eight NTT-friendly primes for the bootstrap modulus chain. None collide with the standard anchor primes.

```rust
pub const BOOTSTRAP_PRIMES: [u64; 8] = [
    998244353,  // 2^23 * 7 * 17 + 1  (work prime 1-3)
    985661441,  // NTT-friendly 30-bit
    754974721,  // NTT-friendly 30-bit
    469762049,  // 2^26 * 7 + 1       (work prime 4, secure_128_deep)
    167772161,  // 2^25 * 5 + 1       (work prime 5, secure_192)
    1811939329, // 27 * 2^26 + 1      (extra for secure_192 modswitch)
    595591169,  // NTT-friendly 30-bit (work prime 6, secure_256)
    645922817,  // NTT-friendly 30-bit (extra for secure_256 modswitch)
];
```

### validate_bootstrap_primes()

Call this to verify bootstrap primes meet FHE requirements:

```rust
use nine65::keys::bootstrap::validate_bootstrap_primes;

validate_bootstrap_primes(&primes, n, target_security_bits)?;
```

Checks:
1. **NTT compatibility**: `(q - 1) % 2N == 0` for all primes
2. **Pairwise coprimality**: `gcd(p_i, p_j) == 1` for all pairs
3. **Security level**: `log2(product) >= target_security`

---

## GaloisKey and GaloisKeySet

For SIMD slot rotations via Galois automorphisms. See the [Batch and Galois](batch-and-galois) page for full details.

```rust
pub struct GaloisKey {
    pub exponent: usize,              // Automorphism exponent k
    pub key_switch_a: Vec<RingPolynomial>,
    pub key_switch_b: Vec<RingPolynomial>,
}

pub struct GaloisKeySet {
    keys: HashMap<usize, GaloisKey>,  // exponent -> key
}
```

---

## Memory Safety

### Zeroize and ZeroizeOnDrop

All secret key types implement `Zeroize` and `ZeroizeOnDrop` from the `zeroize` crate. When a secret key goes out of scope, its memory is overwritten with zeros using volatile writes.

Types with automatic zeroization:
- `SecretKey`
- `DualRNSSecretKey`
- `DualRNSPoly` (since it may hold secret data)
- `RNSSecretKey`

### SecretPoly and SecretScalar

Type-level markers in `security/secret_data.rs` for data requiring constant-time operations:

```rust
use nine65::security::secret_data::{SecretData, SecretPoly, SecretScalar};

// Wrap secret polynomial with automatic zeroization
let secret = SecretPoly::new(sk.s.coeffs.clone(), config.q);
// Automatically zeroized when dropped

// Constant-time comparison
let a = SecretScalar::new(42);
let b = SecretScalar::new(42);
let eq = a.ct_eq(&b); // Returns 1 without branching
```

The `SecretData` trait contract:
1. All operations on this data will use constant-time algorithms
2. Data is zeroized on drop
3. No branching or indexing based on secret values

### Key Lifecycle Manager

`security/key_manager.rs` wraps Clockwork's `KeyLifecycle` for share-based secret key management:

```rust
use nine65::security::key_manager::KeyManager;

let mut mgr = KeyManager::new(config.q, 1000); // reshare every 1000 ops
mgr.initialize(secret_value, randomness)?;

// The full secret key is NEVER stored after initialization.
// Only shares (s1, s2) where s1 + s2 = s (mod q) exist in memory.

// Record operations; reshare when needed
if mgr.record_operation() {
    mgr.reshare(fresh_randomness)?;
}

// Briefly reconstruct for decrypt
let s = mgr.reconstruct_for_decrypt()?;
// ... use s for decryption ...

// Destroy all key material
mgr.destroy();
assert_eq!(mgr.state(), KeyState::Zeroed);
```

---

## Decision Matrix

| Scenario | Key generation | RNG |
|----------|---------------|-----|
| Unit tests | `generate()` | `ShadowHarvester::with_seed(42)` |
| Benchmarks | `generate()` | `ShadowHarvester::with_seed(seed)` |
| CI/KAT | `generate()` | `ShadowHarvester::with_seed(FIXED)` |
| Production encrypt/eval | `generate_secure()` | OS CSPRNG (automatic) |
| Production key ceremony | `generate_secure()` + `KeyManager` | OS CSPRNG |

---

## Where to go next

- [Cookbook](cookbook) -- complete code recipes using these keys
- [Bootstrap System](bootstrap) -- how BootstrapKey and KeySwitchKey are used
- [Batch and Galois](batch-and-galois) -- GaloisKey usage for slot rotations
