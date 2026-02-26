---
layout: default
title: Three-Lock Bootstrap
parent: How It Works
nav_order: 9
---

# Three-Lock Bootstrap

[Home](index)

The Three-Lock Bootstrap is NINE65's defense-in-depth construction for protecting the plaintext during re-encryption. Re-encryption is the moment when the plaintext m must briefly exist in a register, and the Three-Lock system wraps that moment in three orthogonal security barriers.

## Why Three Locks

Bootstrap (re-encryption) is the security-critical operation in any FHE system. The plaintext m must be decrypted from the exhausted ciphertext, then re-encrypted with fresh noise. For one brief instant, m exists unprotected. An attacker with memory access, power analysis, or electromagnetic side-channel equipment could potentially capture it.

The Three-Lock construction ensures that an attacker must break **all three** of the following simultaneously:

```
+-------------------------------------------------------------+
|  SHANNON (Layer 1 -- outermost)                              |
|  Information-theoretic mask over EVERYTHING below             |
|  All memory traces during bootstrap are uniformly random      |
|                                                               |
|  +-----------------------------------------------------------+
|  |  MONTGOMERY (Layer 2 -- middle)                           |
|  |  RLWE encryption around the masked ciphertext             |
|  |  Even if Shannon somehow fails, RLWE must still be broken |
|  |                                                           |
|  |  +-------------------------------------------------------+
|  |  |  CLOCKWORK (Layer 3 -- innermost)                     |
|  |  |  Protected re-encryption                              |
|  |  |  m exists briefly in registers, shielded by both      |
|  |  |  Shannon AND Montgomery                               |
|  |  +-------------------------------------------------------+
|  +-----------------------------------------------------------+
+-------------------------------------------------------------+
```

## The Lock Sequence

### Phase 1: Locking (apply from outside in)

**Layer 1 --- Shannon Mask**: Generate a uniform random polynomial mask using OS CSPRNG. Add it to both ciphertext components:

```
masked_ct = (c0 + r0, c1 + r1)
```

From this point on, all values in memory are uniformly random (Shannon's perfect secrecy). No computational assumption is needed.

**Layer 2 --- Montgomery RLWE Encrypt**: Encrypt the masked ciphertext under a fresh outer secret key:

```
outer_ct = RLWE.Enc(sk_outer, masked_ct)
```

Even if the Shannon mask were somehow compromised, the attacker would still need to break RLWE.

### Phase 2: Unlock + Refresh (peel from outside in)

**Peel Layer 2**: Decrypt the outer RLWE ciphertext. The result is the **still-masked** ciphertext. Shannon is still active.

**Layer 3 --- Clockwork Re-Encryption**: This is the critical inner operation. The Clockwork engine receives:
- The masked ciphertext (c0 + r0, c1 + r1)
- The mask values (r0, r1)
- The secret key s

It performs algebraic mask removal during decryption:

```
Step 1: raw_masked = (c0+r0) + (c1+r1)*s = delta*m + e + (r0 + r1*s)
Step 2: mask_poly = r0 + r1*s              (compute mask contribution)
Step 3: raw_clean = raw_masked - mask_poly = delta*m + e
Step 4: m = decode(raw_clean[0])           (m in register, brief)
Step 5: fresh_ct = encrypt(m)              (re-encrypt with fresh noise)
```

The mask is **never removed from the ciphertext in memory**. The mask contribution is computed from known quantities and subtracted algebraically. The clean plaintext m exists only in CPU registers during steps 3--5.

## Security Tiers

The `SecurityTier` enum controls key rotation policy:

| Tier | Key Rotation | Deployment Model |
|------|-------------|-----------------|
| `Tier1Maximum` | Per-bootstrap outer key rotation | TEE + watchdog + Faraday cage |
| `Tier2Production` | Per-session outer key rotation | Bare metal + TRESOR + IOMMU |
| `Tier3Commodity` | Static outer key | Standard cloud VM |

## Types

### MaskLayer and CiphertextMask (bootstrap/mask.rs)

`MaskLayer` holds a single uniform random polynomial. `CiphertextMask` holds a pair of masks for the two ciphertext components.

Key properties:
- Generated exclusively from OS CSPRNG (`SecureRng`). Shadow entropy must never be used for masks.
- `ZeroizeOnDrop`: masks are securely overwritten when they go out of scope.
- **Mask tracking**: `track_scalar_mul()` and `track_add()` allow the mask to follow linear operations algebraically, maintaining Shannon's guarantee through additions and scalar multiplications.
- Constant-time application: `apply()` and `remove()` are coefficient-wise add/subtract with no data-dependent branches.

### OuterLayer and OuterCiphertext (bootstrap/outer.rs)

`OuterLayer` manages the RLWE encryption for Layer 2. It holds:
- An `OuterSecretKey` (ternary distribution, `ZeroizeOnDrop`)
- Ring dimension, modulus, and error distribution parameter

`OuterCiphertext` is a standard RLWE ciphertext: (b, a) where b = a*s + e + message.

`OuterCiphertextPair` wraps both components of a BFV ciphertext in separate outer encryptions.

`rotate_key()` generates a fresh outer secret key. The old key is zeroized via `ZeroizeOnDrop`.

### ClockworkBootstrap (bootstrap/clockwork.rs)

The innermost engine. Performs protected re-encryption with algebraic mask removal. It holds:
- Ring parameters (n, q, t, eta)
- A `KElimination` context for exact division

`bootstrap_protected()` implements the 5-step algebraic mask removal protocol described above.

`can_bootstrap()` checks whether the noise estimate is within the deterministic decryption threshold Q/(2t).

### ThreeLockBootstrap (bootstrap/three_lock.rs)

The top-level type that orchestrates all three layers:

```rust
let mut tl = ThreeLockBootstrap::new(&config, SecurityTier::Tier2Production);

// Full bootstrap with timing statistics
let (refreshed, stats) = tl.bootstrap(&ct, &bsk, &ntt);

// Fast path without statistics
let refreshed = tl.bootstrap_fast(&ct, &bsk, &ntt);
```

### BootstrapStats

Timing information from each bootstrap execution (all values in nanoseconds, integer-only):

| Field | Description |
|-------|-------------|
| `shannon_mask_ns` | Time to generate and apply Shannon mask |
| `montgomery_encrypt_ns` | Time for outer RLWE encryption |
| `montgomery_decrypt_ns` | Time for outer RLWE decryption |
| `clockwork_ns` | Time for protected re-encryption |
| `total_ns` | Total wall-clock time |
| `key_rotated` | Whether the outer key was rotated after this bootstrap |

## Relationship to ops/bootstrap.rs

The `ops/bootstrap.rs` module provides the higher-level `bootstrap()`, `bootstrap_with_ksk()`, and `AutoBootstrapEvaluator` functions that operate on `DualRNSCiphertext` values. These use the Clockwork bootstrap path for the actual re-encryption. The Three-Lock system wraps Clockwork with the Shannon and Montgomery layers for additional security.

## Where to go next

- [GSO-FHE](gso-fhe) --- the noise bounding system that decides when bootstrap is needed
- [Montgomery Arithmetic](montgomery) --- the constant-time arithmetic that underpins Layer 2
- [Kiosk Architecture](kiosk) --- the self-destructing computation units that complement Three-Lock security
