---
layout: default
title: MANA Accelerator
parent: Using the System
nav_order: 6
---

# MANA Accelerator

[Home](index)

MANA (Modular Anchored Number Arithmetic) is the FHE stream accelerator. It treats CRT prime moduli as parallel compute lanes, enabling Rayon-based parallelism and SIMD-ready coefficient operations.

Source files: `crates/mana/src/lib.rs`, `lane.rs`, `stream.rs`, `anchor.rs`, `gso.rs`, and `nine65/src/accelerated.rs`.

---

## Architecture

```
ManaStream: value V represented as (V mod p1, V mod p2, ..., V mod pk)

Lane 0 [prime=17]: [3, 5, 2, 8, ...]  <- SIMD auto-vectorizable
Lane 1 [prime=19]: [4, 6, 1, 9, ...]  <- Independent, parallel via Rayon
Lane 2 [prime=23]: [1, 2, 0, 5, ...]  <- No carry between lanes
```

Each lane operates on coefficients under a single prime modulus. There are no dependencies between lanes (zero carry propagation), making the structure ideal for parallel execution.

---

## Lane

A single CRT prime channel. All operations are O(N) branchless and SIMD-ready.

```rust
use mana::lane::{Lane, LaneOps};

// Create a lane for prime 998244353 with 1024 coefficients
let lane = Lane::new(998244353, 1024);

// From existing coefficients
let lane = Lane::from_coeffs(vec![1, 2, 3], 998244353);

// From integer values (reduces each mod prime)
let lane = Lane::from_int_slice(&[100, 200, 300], 998244353);

// Properties
let n = lane.len();          // number of coefficients
let p = lane.prime();        // the prime modulus
```

### LaneOps Trait

```rust
pub trait LaneOps {
    fn add(&self, other: &Self) -> Self;       // Coefficient-wise add
    fn sub(&self, other: &Self) -> Self;       // Coefficient-wise sub
    fn neg(&self) -> Self;                     // Negate all coefficients
    fn mul(&self, other: &Self) -> Self;       // Hadamard product (element-wise)
    fn scalar_mul(&self, scalar: u64) -> Self; // Scalar multiply all
    fn scalar_add(&self, scalar: u64) -> Self; // Add scalar to all
}
```

All arithmetic uses branchless modular operations designed for LLVM auto-vectorization. Add/sub use a conditional-move pattern: compute `a + b`, then subtract the prime if the result overflowed. No branches, no carry propagation.

Lane implements `Zeroize` for secure memory clearing when handling sensitive data.

---

## ManaStream

Multi-lane CRT representation. Each lane holds coefficients under one prime. The primes vector is shared via `Arc<Vec<u64>>` to avoid clone overhead.

```rust
use mana::stream::{ManaStream, StreamOps};

// Create a stream with 3 primes, 1024 coefficients per lane
let primes = &[998244353u64, 985661441, 754974721];
let stream = ManaStream::new(primes, 1024);

// From integer values -- each value is reduced to all primes
let values = vec![12345u64; 1024];
let stream = ManaStream::from_ints(&values, primes);

// Properties
let num_lanes = stream.num_lanes(); // 3
let n = stream.len();               // 1024
let product = stream.product();     // cached product of all primes (u128)

// CRT reconstruction at position i
let value = stream.reconstruct_at(0); // recovers the original integer
```

### StreamOps Trait

```rust
pub trait StreamOps {
    fn add(&self, other: &Self) -> Self;
    fn sub(&self, other: &Self) -> Self;
    fn neg(&self) -> Self;
    fn mul(&self, other: &Self) -> Self;
    fn scalar_mul(&self, scalar: u64) -> Self;
}
```

Stream operations apply the corresponding `LaneOps` operation to each lane. With the `parallel` feature, these dispatch across lanes using Rayon.

---

## AnchorContext and KAnchor

K-Elimination exact division within MANA.

### KAnchor

```rust
use mana::anchor::KAnchor;

// Create anchor with alpha (computational) and beta (anchor) codex primes
let anchor = KAnchor::new(
    &[65537, 65521, 65519],           // alpha primes
    &[4611686018427387847u64],        // beta primes
);

// Prebuilt for FHE operations (~110-bit total capacity)
let anchor = KAnchor::for_fhe();

// Extract k value for exact reconstruction
let k = anchor.extract_k(v_alpha, v_beta);
// V = v_alpha + k * alpha_cap
```

Fields:
- `alpha_primes`, `beta_primes`: the two codex prime sets
- `alpha_cap`, `beta_cap`: products of each set
- `alpha_inv_beta`: precomputed `alpha_cap^-1 mod beta_cap`

### AnchorContext

Higher-level wrapper around `KAnchor` for use in stream operations.

```rust
use mana::anchor::AnchorContext;

let anchor_ctx = AnchorContext::for_fhe();
```

---

## GsoSwarm and QbitAgent

Glowworm Swarm Optimization with quantum-inspired operations, implemented within MANA for FHE parameter search and noise optimization.

### QbitState

A quantum-inspired superposition of search states represented as CRT residues.

```rust
use mana::gso::QbitState;

// Uniform superposition over 8 states
let qbit = QbitState::uniform(8, 998244353);

// From explicit weights
let qbit = QbitState::from_weights(vec![1, 2, 3, 1, 1, 1, 1, 1], 998244353);

// Probability of state i (as integer ratio)
let (num, den) = qbit.probability(2);

// Rotate amplitude toward a target state (like Grover's amplification)
qbit.rotate_toward(target_state, rotation_amount);

// Apply Hadamard-like mixing (spreads amplitude)
qbit.mix();
```

### GsoSwarm

The full swarm with agents, luciferin updates, and movement.

```rust
use mana::gso::GsoSwarm;

let swarm = GsoSwarm::new(n_agents, search_radius);
```

Each agent evolves through:
1. **Luciferin update**: brightness based on fitness function
2. **Movement**: probabilistic movement toward brighter neighbors
3. **Qbit rotation**: amplitude adjustment toward promising states

All operations are integer-only. No floating-point anywhere.

---

## ParallelStream

Available with the `parallel` feature flag. Executes lane operations across multiple threads via Rayon.

```rust
// Cargo.toml
// [dependencies]
// mana = { path = "../mana", features = ["parallel"] }

use mana::parallel::ParallelStream;
```

Benchmark result: ~2.78x speedup on 4-lane RNS operations with Rayon parallelism.

---

## AcceleratedFHE (Integration Point)

Defined in `nine65/src/accelerated.rs`. This is the bridge between NINE65's FHE operations and MANA's parallel streams. Requires the `accelerated` feature flag.

```rust
use nine65::accelerated::AcceleratedFHE;

// Create accelerated context (auto-detects hardware via UNHAL)
let fhe = AcceleratedFHE::new(&config);

// Convert between RNS polynomials and MANA streams
let stream = fhe.rns_to_stream(&rns_polynomial);
let rns = fhe.stream_to_rns(&stream);

// Accelerated ciphertext operations
let ct_sum = fhe.add_ciphertexts(&ct_a, &ct_b);
```

### How It Works

1. `AcceleratedFHE::new()` creates an `Accelerator` (from UNHAL) and an `AnchorContext` (from MANA)
2. RNS polynomial limbs are mapped 1:1 to MANA lanes
3. Stream operations execute in parallel across lanes
4. Results are mapped back to RNS polynomials

### Conversion

```rust
// RNS -> MANA: each RNS limb becomes a MANA lane
fn rns_to_stream(&self, rns: &RNSPolynomial) -> ManaStream

// MANA -> RNS: each lane's coefficients become an RNS limb
fn stream_to_rns(&self, stream: &ManaStream) -> RNSPolynomial
```

---

## When to Use MANA vs Raw RNSFHEContext

| Scenario | Use |
|----------|-----|
| Single operations, small N | `RNSFHEContext` directly |
| Batch operations, large N (4096+) | `AcceleratedFHE` with MANA |
| Parameter search / optimization | `GsoSwarm` |
| Custom parallel pipeline | `ManaStream` with `parallel` feature |
| Embedded / no-alloc targets | `Lane` directly |

MANA adds overhead for small problems. The crossover point where parallelism pays off is around N=2048 with 3+ CRT lanes.

---

## Feature Flags

| Flag | Effect |
|------|--------|
| `parallel` | Enables Rayon-based parallel lane processing |
| `accelerated` | Enables `AcceleratedFHE` in nine65 (requires mana + unhal) |

---

## Where to go next

- [Neural Operations](neural-ops) -- neural workloads that benefit from MANA acceleration
- [Entropy](entropy) -- entropy harvesting from MANA stream byproducts
- [Architecture](architecture) -- how MANA fits into the overall system
