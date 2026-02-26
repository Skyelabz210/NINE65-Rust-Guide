---
layout: default
title: Glossary
parent: Foundation
nav_order: 7
---

# Glossary

[Home](index)

Terms used throughout the NINE65 codebase and documentation. Each entry gives a one-sentence definition and points to the relevant page or source file.

---

### Basin
{: #basin }

An attractor region in GSO-FHE noise space; each plaintext value maps to a basin, and when noise exceeds the basin radius a [collapse](#collapse) is triggered to reset it. See `crates/nine65/src/ops/gso_fhe.rs`.

### BFV
{: #bfv }

Brakerski/Fan-Vercauteren --- the FHE encryption scheme implemented by NINE65, operating on polynomial rings with integer-only arithmetic. See [Architecture](architecture).

### BKZ
{: #bkz }

Block Korkine-Zolotarev --- a lattice reduction algorithm used to estimate the hardness of breaking RLWE-based encryption; the primary attack model for [security estimation](security-configs).

### Bootstrap
{: #bootstrap }

A procedure that takes a noisy ciphertext and produces a fresh ciphertext encrypting the same plaintext with reset noise, enabling unlimited-depth computation. NINE65 implements three paths: circular, non-circular KSK, and auto-triggered. See [Bootstrap](bootstrap).

### BSK
{: #bsk }

Bootstrap Secret Key --- an encrypted form of the secret key used during the bootstrap procedure to evaluate the decryption circuit homomorphically. See `crates/nine65/src/keys/bootstrap.rs`.

### Bullet
{: #bullet }

A [Kiosk](#kiosk) unit type that self-destructs after a single computation. Created via `KioskUnit::bullet()`. See `crates/nine65/src/kiosk/unit.rs`.

### Capsule
{: #capsule }

A [Kiosk](#kiosk) unit type that self-destructs after a budgeted number of operations. Created via `KioskUnit::capsule(id, moduli, seed, max_ops)`. See `crates/nine65/src/kiosk/unit.rs`.

### CBD
{: #cbd }

Centered Binomial Distribution --- the noise distribution used for error sampling during encryption, parameterized by eta (e.g., eta=3 for `secure_128`). Values are drawn in the range `[-eta, eta]`. See `FheRng::cbd()`.

### Ciphertext
{: #ciphertext }

An encrypted value, represented as a pair of polynomials `(c0, c1)` in the ring `R_q`. Noise increases with each homomorphic operation. See [Noise Budget](noise-budget).

### Collapse
{: #collapse }

In GSO-FHE, the operation that resets a ciphertext's noise estimate back to zero when it exceeds the [basin](#basin) radius. Analogous to bootstrap but modeled as swarm reconvergence. See `NoiseEstimate::collapse()` in `gso_fhe.rs`.

### Constant-time
{: #constant-time }

Code whose execution time does not depend on secret data, preventing timing side-channel attacks. Enforced in NINE65 security-critical paths using the `subtle` crate for constant-time comparisons and the `U256` type for constant-time arithmetic. See [Rust Patterns](rust-patterns).

### Core-SVP
{: #core-svp }

Core Shortest Vector Problem --- the cost model used by the NINE65 lattice security estimator to estimate classical and quantum attack costs. See [Security Configs](security-configs).

### CRT
{: #crt }

Chinese Remainder Theorem --- the mathematical foundation for representing a large integer as a tuple of residues modulo coprime moduli, enabling parallel arithmetic. Core to the [RNS](#rns) representation used throughout NINE65. See [Architecture](architecture).

### Cyclotomic
{: #cyclotomic }

Refers to cyclotomic polynomial rings `Z[X]/(X^n + 1)` where n is a power of 2. All NINE65 polynomial arithmetic operates in these rings. The `CyclotomicRing` and `CyclotomicPolynomial` types live in `crates/nine65/src/arithmetic/cyclotomic_phase.rs`.

### Delta
{: #delta }

The scaling factor `floor(Q/t)` used in BFV encoding. A plaintext `m` is encoded as `m * delta` before adding noise. Correct decryption requires `delta` to be large relative to noise. See `RNSFHEContext::delta_rns`.

### DualRNS
{: #dualrns }

The two-track RNS architecture: main primes for computation plus anchor primes for [K-Elimination](#k-elimination). Ciphertexts carry residues in both tracks from the moment of encryption. See `DualRNSContext` and `DualRNSCiphertext` in `rns_fhe.rs`.

### Evaluation key
{: #evaluation-key }

A public key component containing encrypted values of s-squared, enabling relinearization after ciphertext multiplication without the secret key. See `DualRNSEvalKey` and `EvaluationKey`.

### Fuse
{: #fuse }

A [Kiosk](#kiosk) unit type that self-destructs after a time limit (measured in nanoseconds). Created via `KioskUnit::fuse(id, moduli, seed, lifetime_ns)`. See `crates/nine65/src/kiosk/unit.rs`.

### Galois automorphism
{: #galois-automorphism }

A ring automorphism `X -> X^k` that permutes SIMD slots in a batched ciphertext. Used for slot rotations in encrypted vector operations. See `GaloisEngine` and `GaloisEvaluator` in the prelude.

### GRO
{: #gro }

Galois Ring Oscillator --- a timing gate mechanism in the Clockwork subsystem that provides deterministic bound tracking and key lifecycle integrity. Enabled by the `clockwork` feature flag. See [Feature Flags](feature-flags).

### GSO
{: #gso }

Glowworm Swarm Optimization --- the noise management strategy used in `GSOFHEContext` where noise is modeled as distance from a basin center, and collapse (reconvergence) replaces traditional bootstrapping. See `crates/nine65/src/ops/gso_fhe.rs`.

### HE Standard
{: #he-standard }

The Homomorphic Encryption Standard (v1.1) --- industry guidelines for minimum parameter sizes at each security level. NINE65's `SecureConfig` constructors verify compliance. See [Security Configs](security-configs).

### K-Elimination
{: #k-elimination }

The exact integer division algorithm that solves the 70-year RNS division bottleneck. Uses anchor primes to compute `k = (x_R - x_P) * C_P^(-1) mod C_R`, then eliminates the quotient factor to recover the exact result. See `crates/nine65/src/arithmetic/k_elimination.rs` and the Coq proof in `proofs/coq/KElimination.v`.

### Kiosk
{: #kiosk }

Self-destructing FHE computation units with enforced lifecycle constraints. Three types: [Bullet](#bullet) (single-use), [Capsule](#capsule) (budget-limited), [Fuse](#fuse) (time-limited). See `crates/nine65/src/kiosk/`.

### KSK
{: #ksk }

Key Switching Key --- used in non-circular bootstrap to switch from a boot-time secret key to the working secret key without exposing either. See `crates/nine65/src/keys/bootstrap.rs`.

### Lattice
{: #lattice }

A discrete subgroup of R^n. The security of RLWE-based FHE (including NINE65) relies on the hardness of finding short vectors in lattices. See [Security Configs](security-configs).

### MATZOV
{: #matzov }

A refined lattice attack cost model from the MATZOV group (2022) that gives tighter estimates than Core-SVP for some parameter ranges. The NINE65 security estimator reports both Core-SVP and MATZOV figures. See `docs/LATTICE_ESTIMATOR_BASELINE_2026-02-25.md`.

### Millibits
{: #millibits }

The unit of noise measurement in NINE65: 1000 millibits = 1 bit. Using integer millibits instead of floating-point log2 values avoids rounding errors in noise tracking. See `NoiseBudget` in `crates/nine65/src/noise/budget.rs`.

### Modswitch
{: #modswitch }

Modulus switching --- reducing a ciphertext's modulus from Q to a smaller Q' by scaling and rounding, which proportionally reduces noise at the cost of one prime. See `RNSFHEContext::mod_switch_ct_to_level()`.

### Montgomery form
{: #montgomery-form }

A representation where a value `a` is stored as `a * R mod q` (where R is a power of 2), enabling division-free modular multiplication. NINE65 keeps values in Montgomery form persistently to avoid repeated conversions. See `MontgomeryContext` and `PersistentMontgomery` in the prelude.

### MQ-ReLU
{: #mq-relu }

Modular-Quotient Rectified Linear Unit --- an integer-only activation function for encrypted neural networks. Applies `f(x) = x if x < threshold, else 0` using modular arithmetic, no floats. See `MQReLU` in `crates/nine65/src/arithmetic/mq_relu.rs`.

### Nine65Result
{: #nine65result }

The `Result<T, Nine65Error>` type alias used across the codebase. See [Rust Patterns](rust-patterns).

### Noise budget
{: #noise-budget }

The remaining capacity for operations before decryption fails. Measured in [millibits](#millibits). Each add costs ~1 millibit; each multiply costs thousands. When budget reaches zero, the ciphertext is garbage. See [Noise Budget](noise-budget).

### NTT
{: #ntt }

Number Theoretic Transform --- the integer analogue of FFT, operating modulo a prime. Converts polynomial multiplication from O(n^2) to O(n log n). See `NTTEngine` / `NTTEngineFFT` in the prelude.

### NTT-friendly prime
{: #ntt-friendly-prime }

A prime `p` such that `p = 1 (mod 2n)`, guaranteeing the existence of primitive 2n-th roots of unity needed for negacyclic NTT. All primes in NINE65 configs (998244353, 985661441, 754974721, 469762049, 167772161, 595591169) are NTT-friendly.

### Pade
{: #pade }

Pade approximation engine --- computes rational function approximations of transcendental functions using integer-only scaled arithmetic. See `PadeEngine` in `crates/nine65/src/arithmetic/pade_engine.rs`.

### Permille
{: #permille }

Parts per thousand (notation: per mille). Used throughout NINE65 instead of percentages for threshold comparisons because permille values are integers (e.g., 900 permille = 90%), avoiding float arithmetic.

### Plaintext modulus
{: #plaintext-modulus }

The value `t` that defines the plaintext space `Z_t`. All NINE65 production configs use `t = 65537`. Plaintexts must satisfy `0 <= m < t`. See [Security Configs](security-configs).

### Polynomial ring
{: #polynomial-ring }

The quotient ring `Z_q[X]/(X^n + 1)` where ciphertexts and keys live. The degree `n` determines the ring dimension (4096 or 16384 for production configs). See [Architecture](architecture).

### Public key
{: #public-key }

A pair of polynomials `(pk0, pk1)` derived from the secret key, used for encryption. Anyone with the public key can encrypt; only the secret key holder can decrypt. See `PublicKey` and `DualRNSPublicKey`.

### Relinearization
{: #relinearization }

The process of reducing a degree-2 ciphertext (produced by multiplication) back to degree 1 using an [evaluation key](#evaluation-key). Without relinearization, ciphertext size grows with each multiply.

### Rescaling
{: #rescaling }

In NINE65, the K-Elimination-based exact division that removes a prime from the ciphertext modulus after multiplication, keeping noise bounded. Unlike approximate rescaling in CKKS, this is exact. See [K-Elimination](#k-elimination).

### RLWE
{: #rlwe }

Ring Learning With Errors --- the computational hardness assumption underlying BFV security. Given a polynomial ring element that is either random or structured (secret + small error), distinguishing the two is believed to be hard. See [Security Configs](security-configs).

### RNS
{: #rns }

Residue Number System --- representing a value as a tuple of residues modulo coprime primes. Enables coefficient-wise parallel arithmetic with zero carry propagation between lanes. See [Architecture](architecture).

### Secret key
{: #secret-key }

A ternary polynomial `s` with coefficients in {-1, 0, 1}. Required for decryption. Implements `Zeroize` and `ZeroizeOnDrop` for secure memory clearing. See `SecretKey` and `DualRNSSecretKey`.

### SIMD
{: #simd }

Single Instruction, Multiple Data --- in the FHE context, packing multiple plaintext values into a single ciphertext's slots via CRT batching. Enabled when `t = 1 (mod 2n)`. See `BatchEncoder`.

### Ternary secret
{: #ternary-secret }

A secret key distribution where each coefficient is drawn from {-1, 0, 1} with equal probability. Provides a meet-in-the-middle attack surface accounted for in hybrid security estimates.

### Three-Lock
{: #three-lock }

The conjunction security model used in symmetric bootstrap: Shannon entropy masking + RLWE encryption + timing isolation (Clockwork). All three must be broken simultaneously for an attack to succeed. See `crates/nine65/src/ops/symmetric_bootstrap.rs`.

### U256
{: #u256 }

A crate-internal (`pub(crate)`) 256-bit unsigned integer type (`lo: u128`, `hi: u128`) used for intermediate CRT reconstruction when products exceed u128 range. See `crates/nine65/src/arithmetic/rns.rs`.

### Zeroize
{: #zeroize }

The `zeroize` crate's trait for securely overwriting memory with zeros using volatile writes that the compiler cannot optimize away. Applied to all secret key types in NINE65. See [Rust Patterns](rust-patterns).

---

## Where to go next

- [Architecture](architecture) --- how these concepts fit together into the full system
- [Rust Patterns](rust-patterns) --- the code patterns behind types like `Nine65Result`, `FheRng`, and `ZeroizeOnDrop`
- [Security Configs](security-configs) --- the production parameter sets with verified security levels
