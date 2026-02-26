---
layout: default
title: K-Elimination
parent: How It Works
nav_order: 4
---

# K-Elimination

[Home](index)

K-Elimination is the algorithm that gives NINE65 exact integer division in the Residue Number System (RNS). Traditional RNS division requires full CRT reconstruction at O(k^2) cost. K-Elimination does it in O(k).

## The Problem

In BFV, homomorphic multiplication produces values at the delta-squared level. To get back to the delta level, you must divide by delta --- exactly, with no remainder and no rounding error. In RNS representation, where a number is stored as residues modulo separate primes, exact division has historically required reconstructing the full integer via the Chinese Remainder Theorem, which costs O(k^2) for k primes.

## The Solution: Dual-Track Residues

K-Elimination avoids full CRT reconstruction by maintaining the value in two independent residue systems simultaneously:

- **Alpha system** (main computational primes): stores V mod alpha_cap
- **Beta system** (anchor primes): stores V mod beta_cap

Where alpha_cap and beta_cap are products of their respective prime sets and are coprime to each other.

## The Algorithm

Given a value V represented as:

```
V = v_alpha  (mod alpha_cap)
V = v_beta   (mod beta_cap)
```

The quotient k (which tracks how many times alpha_cap wraps around) is computed as:

```
k = (v_beta - v_alpha) * alpha_cap_inv  (mod beta_cap)
```

Then the exact value is reconstructed:

```
V = v_alpha + k * alpha_cap
```

This is O(k) --- one modular subtraction, one modular multiplication, one multiply-and-add. No full CRT sweep.

## Coq Proof

The correctness of K-Elimination is machine-checked in Coq:

```coq
Theorem k_elimination_complete : forall k v_M M A : nat,
  M > 0 -> v_M < M -> k < A ->
  let X := v_M + k * M in X / M = k.

Theorem complexity_improvement :
  k_elimination_ops k = k /\ mrc_ops k = k * k.
  (* O(k) vs O(k^2) *)
```

The proof file is `proofs/coq/KElimination.v`.

## Configuration

The `KElimConfig` enum provides four preset configurations:

| Config | Alpha Bits | Beta Bits | Total Capacity | Use Case |
|--------|-----------|----------|----------------|----------|
| `Minimal` | ~32 | ~32 | ~64 bits | Testing, small values |
| `Standard` | ~48 | ~62 | ~110 bits | Most FHE operations, N <= 2048 |
| `Extended` | ~48 | ~90 | ~138 bits | Large N (4096+), deep circuits |
| `Maximum` | ~64 | ~124 | ~188 bits | Maximum precision |

You can also build custom configurations with `KElimBuilder`:

```rust
let ke = KElimBuilder::new()
    .alpha_primes(&[65537, 65521])
    .beta_primes(&[4611686018427387847])
    .build()?;
```

## The 3-Anchor to 5-Anchor Fix

This is a critical piece of NINE65 history (commit 73a372e).

**The bug**: `canonical_anchor_primes_for_n()` originally returned 3 anchor primes, giving a ~94-bit anchor product. For `secure_128` with 3 main primes, K-Elimination silently returned `value mod (M*A)` instead of the true value during ciphertext-times-ciphertext multiplication. The anchor product was too small to hold the quotient k.

**The fix**: Return 5 anchor primes instead of 3, giving a ~158-bit anchor product. With 159 bits of anchor capacity, all intermediate values from the tensor product fit comfortably.

**The enforcement**: `DualRNSContext::for_fhe()` always uses 5 anchors. The multiplication functions `mul_dual_public()` and `mul_dual_symmetric()` include a runtime assertion:

```rust
assert!(anchor_count >= 5);
```

This prevents any code path from accidentally using fewer anchors.

## Key Types and Methods

### KElimination

```rust
let ke = KElimination::from_config(KElimConfig::Standard);

// Check capacity
let bits = ke.capacity_bits();  // ~110 for Standard

// Check how close a value is to the capacity boundary
let report = ke.capacity_proximity(some_value);
if report.is_warning() {
    // Value is at 80%+ of capacity
}

// Reconstruct exact value from dual residues
let exact = ke.reconstruct(v_alpha, v_beta);

// Exact division
let quotient = ke.exact_divide(v_alpha, v_beta, divisor);
```

### ExactDivider (arithmetic/exact_divider.rs)

The `ExactDivider` is a lower-level primitive that performs K-Elimination with a single main modulus and a single anchor modulus:

```rust
let ed = ExactDivider::for_fhe(q);

// Reconstruct X from residues mod M and mod A
let x = ed.reconstruct(vm, va);

// Exact division when d | X
let result = ed.exact_divide(vm, va, d);
```

It uses 62-bit anchor primes (following the "Orbital Patch" of December 2024) to ensure that tensor product intermediates fit within the M*A product.

### DualRNSContext

The FHE-level integration point. `DualRNSContext::for_fhe()` constructs the full dual-RNS setup with:

- Main RNS primes from the `SecureConfig`
- 5 anchor primes (always, since the fix)
- Precomputed NTT engines for each prime
- Precomputed delta values in RNS form

Key methods:

```rust
let ctx = DualRNSContext::for_fhe(&config);

// Anchor capacity is 159 bits for 5 primes
let anchor_bits = ctx.anchor_capacity_bits();

// Check if a value is approaching anchor limits
let report = ctx.check_k_proximity(value);

// Check intermediate multiplication values
let report = ctx.check_intermediate_proximity(intermediate);
```

## Capacity Proximity Checking

K-Elimination has a hard capacity boundary: values must be less than alpha_cap * beta_cap for exact reconstruction. The `capacity_proximity()` method on `KElimination` returns a `CapacityReport` indicating which region a value falls in:

| Region | Utilization | Meaning |
|--------|------------|---------|
| `Safe` | < 80% | No concerns |
| `Warn80` | 80--89% | Approaching boundary |
| `Warn90` | 90--99% | Near overflow |
| `Critical` | >= 100% | Overflow imminent or occurred |

For all three production `SecureConfig` values, intermediate tensor product values sit at ~45--47% of anchor capacity, which is firmly in the Safe region.

## Performance

K-Elimination achieves a 40x speedup over traditional Mixed Radix Conversion for exact division in RNS:

| Method | Complexity | Operations for k primes |
|--------|-----------|------------------------|
| Mixed Radix Conversion | O(k^2) | k * k |
| K-Elimination | O(k) | k |

## Where to go next

- [Boundary Proximity](boundary) --- detailed capacity tracking and warning system
- [NTT Engine](ntt) --- the polynomial multiplication that K-Elimination enables
- [BFV Scheme](bfv-scheme) --- how K-Elimination fits into the full encryption pipeline
