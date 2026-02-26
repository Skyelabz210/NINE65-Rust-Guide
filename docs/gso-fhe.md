---
layout: default
title: GSO-FHE
parent: How It Works
nav_order: 10
---

# GSO-FHE

[Home](index)

Gravitational Swarm Optimization FHE (GSO-FHE) provides a noise bounding model for NINE65's homomorphic operations. It tracks cumulative noise growth and triggers basin collapse when noise exceeds a threshold --- a lightweight noise management layer that complements the full Clockwork Bootstrap for noise refresh.

## The Noise Problem

Every homomorphic operation increases the noise in a ciphertext. Addition increases noise linearly. Multiplication increases noise multiplicatively (roughly squaring it). Eventually, if noise exceeds the decryption threshold delta/2, decryption fails silently --- the wrong plaintext is recovered.

Traditional BFV systems handle this by bootstrapping (re-encrypting) periodically, which costs 100--1000ms per operation.

## GSO Approach

GSO-FHE models the plaintext space as a set of **attractor basins**. Each plaintext value maps to a basin with a defined radius. Noise growth is tracked as distance from the basin center. When the distance exceeds the basin radius, a **collapse** operation reconverges the swarm --- a lightweight reset that costs approximately 1ms versus the 100--1000ms of a full bootstrap.

The key concepts:

### NoiseEstimate

Tracks cumulative noise for a single ciphertext:

```rust
pub struct NoiseEstimate {
    pub distance: u64,        // Cumulative noise distance from basin center
    pub basin_id: u32,        // Basin this ciphertext belongs to
    pub mul_depth: u32,       // Number of multiplications performed
    pub collapse_count: u32,  // Number of collapses performed
}
```

Noise updates follow the BFV noise model:
- **After addition**: `distance = distance_a + distance_b` (linear growth)
- **After multiplication**: `distance = d_a + d_b + (d_a * d_b) / coefficient_bound` (multiplicative growth from tensor product cross-terms)

### AttractorBasin

Each basin represents a region in state space where noise is bounded:

```rust
pub struct AttractorBasin {
    pub id: u32,
    pub center_x: i64,
    pub center_y: i64,
    pub radius: u64,
}
```

Basins are placed using **golden-angle spacing** computed with Q30 fixed-point arithmetic and a Q15 cosine/sine lookup table. This gives uniform coverage of the state space without any floating-point operations.

### GSOSwarm

The swarm is a simplified gravitational swarm optimizer that performs basin collapse:

```rust
pub struct GSOSwarm {
    pub n_agents: usize,
    pub target: Option<AttractorBasin>,
    pub convergence_threshold: u64,
    pub step: u64,
    shadow: u64,  // Shadow entropy accumulator
}
```

During collapse, the swarm runs deterministic dynamics for `log2(radius) + 10` iterations. A useful side effect: the dynamics produce **shadow entropy** (deterministic mixing of basin coordinates), which can be extracted for auxiliary randomness.

### GSOCiphertext

Wraps a `DualRNSCiphertext` with noise tracking:

```rust
pub struct GSOCiphertext {
    pub inner: DualRNSCiphertext,
    pub noise: NoiseEstimate,
}
```

## GSOFHEContext

The main interface for GSO-tracked FHE operations:

```rust
let inner = RNSFHEContext::new(&config);
let gso = GSOFHEContext::new(inner);

// Generate keys
let keys = gso.generate_keys_secure();

// Encrypt with noise tracking
let ct = gso.encrypt(42, &keys.public_key, &mut rng);

// Homomorphic operations with automatic noise tracking
let sum = gso.add(&ct1, &ct2);       // Noise grows linearly
let prod = gso.mul_symmetric(&ct1, &ct2, &keys.secret_key);  // Noise grows multiplicatively

// Check noise status
if ct.noise.needs_collapse(gso.basin_radius) {
    // Collapse triggered automatically in mul operations
}

// Decrypt (noise tracking does not affect decryption)
let m = gso.decrypt(&ct, &keys.secret_key);
```

### Basin Radius Computation

The basin radius is derived from the scaling factor delta:

```
basin_radius = delta / 4
```

Where delta = q_product / t. This gives the noise threshold at which collapse is triggered --- roughly one-quarter of the maximum tolerable noise.

The coefficient bound used for multiplicative noise estimation is `isqrt(q_product)` (integer square root of the ciphertext modulus product).

### Key Generation

`GSOFHEContext` delegates key generation to the underlying `RNSFHEContext`:

| Method | Description |
|--------|-------------|
| `generate_keys(rng)` | Symmetric keys with provided RNG |
| `generate_keys_secure()` | Symmetric keys with OS CSPRNG |
| `generate_full_keys(rng)` | Full keys including eval key (for public multiplication) |
| `generate_full_keys_secure()` | Full keys with OS CSPRNG |
| `generate_full_keys_public_deep(rng)` | Full keys with smaller decomposition base for deeper circuits |

## How GSO Relates to Bootstrap

GSO-FHE and Clockwork Bootstrap serve complementary roles:

| Aspect | GSO Basin Collapse | Clockwork Bootstrap |
|--------|-------------------|-------------------|
| **Purpose** | Noise tracking and lightweight bounding | Full noise refresh via re-encryption |
| **Cost** | ~1ms (swarm convergence) | Variable (depends on bootstrap path) |
| **Noise model** | Tracks cumulative distance from basin center | Resets to fresh encryption noise |
| **When used** | Per-operation noise estimation | When noise budget is exhausted |
| **Depth** | Bounded by basin radius | Unlimited (re-encryption gives fresh ciphertext) |

In practice, the GSO noise tracker monitors noise growth. When it detects that a ciphertext is approaching the basin boundary, it can trigger a full bootstrap through the Three-Lock system to refresh the noise budget completely.

## Depth Benchmarks

The test `benchmark_symmetric_max_depth_secure_128` measures how many consecutive multiplications can be performed before decryption fails. Run it with:

```bash
cargo test -p nine65 --lib --release \
    ops::gso_fhe::depth_benchmarks::benchmark_symmetric_max_depth_secure_128 \
    -- --nocapture
```

## Where to go next

- [Three-Lock Bootstrap](three-lock) --- the full re-encryption system triggered when noise is exhausted
- [Boundary Proximity](boundary) --- the capacity tracking system for K-Elimination values
- [Noise Budget](noise-budget) --- the noise budget tracking system in the basic BFV evaluator
