---
layout: default
title: Clockwork Core
parent: Tools & Testing
nav_order: 5
---

# Clockwork Core

[Home](index)

`clockwork-core` is a standalone Rust crate that implements the Loki-Clockwork Formal Specification v1.0. It provides exact, integer-only RNS (Residue Number System) arithmetic with bound tracking, GRO timing gates, triple-redundant integrity, and key lifecycle management. The crate has 46 tests and zero floating-point operations.

**Source**: `crates/clockwork-core/src/`

---

## What clockwork-core is

Clockwork-core is the formal-spec compliant arithmetic foundation that sits underneath `nine65`'s FHE operations. While `nine65::arithmetic` provides the K-Elimination and NTT implementations optimized for FHE ciphertext processing, `clockwork-core` provides the reference RNS types with mathematically proven invariants. Think of it as the spec-aligned layer that guarantees correctness, while `nine65::arithmetic` is the performance-optimized layer that implements the same algorithms at scale.

---

## Design invariants

The crate enforces seven invariants from the formal specification (section 3):

| ID | Invariant | Enforcement |
|----|-----------|-------------|
| **INV-1** | RLWE modulus `q` is fixed. No operation modifies it. | `DecodeToQ` stores `q` as an immutable field. No setter exists. |
| **INV-2** | Round-trip through Clockwork residues is the identity on Z_q. | `DecodeToQ::verify_roundtrip` tests this for all `c < q`. Tested exhaustively in CT-05. |
| **INV-3** | Bound tracker is always sound: `|Center(X)| < 2^H(X)`. | `GearStack` derives bounds from operation rules (D7), never from actual values. Verified in CT-03. |
| **INV-4** | GRO window sizes are schedule-determined. | `GroGate` computes windows from phase increments, not from runtime data. |
| **INV-5** | Full secret key `s` is never stored after SPLIT phase. | `KeyLifecycle` state machine enforces this: `initialize()` transitions through SPLIT to ACTIVE, where only `(s1, s2)` exist. |
| **INV-7** | Operation schedule is public and value-independent. | Bound updates use `Bound::add_bound`, `Bound::mul_bound` etc., which depend only on operand bounds, not values. |
| **INV-8** | Triple-redundant tier state must pass `MajVote` before any `DecQ` operation. | `TripleRedundant::read()` returns `VoteResult::Failed` on double corruption, triggering fail-closed (T15). |

---

## Modules

### basis (RnsBasis) -- Formal Spec D1-D4

**File**: `basis.rs`

The foundational type. `RnsBasis` holds a vector of pairwise coprime moduli, their product (capacity `M_k`), and precomputed CRT coefficients.

Key operations:
- `RnsBasis::new(moduli)` -- validates pairwise coprimality, computes capacity and CRT coefficients. Returns `BasisError` on failure.
- `encode(x)` -- D2: Enc_p(X) = (X mod p_0, ..., X mod p_k).
- `encode_signed(x)` -- handles negative values via modular arithmetic.
- `decode(residues)` -- D3: CRT reconstruction to the unique X in [0, M_k).
- `centered_lift(x)` -- D4: Center_{M_k}(x) in [-M_k/2, M_k/2].
- `extend(new_modulus)` -- D8: basis promotion by appending a coprime modulus.
- `promote_verified(residues, new_modulus, claimed_residue)` -- checked promotion with consistency verification.
- `demote(residues, target_len)` -- restrict to first k moduli (T3 guarantee).

All u128 multiplication uses a shift-and-add approach to avoid overflow. Zero floating-point.

Theorems covered: T1 (CRT bijectivity, tested exhaustively in CT-01), T3 (promotion/demotion roundtrip, CT-04), T5 (DecQ roundtrip, CT-05).

### bound_tracker (Bound) -- Formal Spec D7

**File**: `bound_tracker.rs`

Deterministic integer bounds on the magnitude of centered-lift values. Replaces all floating-point `log2_height` estimates with exact, auditable, schedule-based bounds.

Key property (T4 -- Bound Tracker Soundness): if all input values satisfy `|Center(X)| < 2^H(X)`, then all computed values satisfy `|Center(Y)| < 2^H(Y)` under these rules:

| Operation | Bound rule |
|-----------|-----------|
| Addition | `H(X+Y) <= max(H(X), H(Y)) + 1` |
| Subtraction | `H(X-Y) <= max(H(X), H(Y)) + 1` |
| Multiplication | `H(X*Y) <= H(X) + H(Y)` |
| Dot product | `H(sum w_j*x_j) <= ceil(log2(L)) + max_j(H(w_j) + H(x_j))` |
| Negation | `H(-X) = H(X)` |
| Scalar multiply | `H(c*X) <= H(c) + H(X)` |

The `Bound` type stores a single `u32` exponent. `needs_promotion(capacity_bits, guard)` implements the D7 promotion policy: `M_k > 2^(H + guard)`.

### decode_to_q (DecodeToQ) -- Formal Spec D9

**File**: `decode_to_q.rs`

The bridge from Clockwork representation space (Z_{M_k}) back to RLWE ciphertext space (Z_q). This is where INV-1 and INV-2 are enforced.

- `DecodeToQ::new(q, convention)` -- creates the bridge with a FIXED modulus. Supports `Unsigned` or `Centered` decode conventions.
- `decode(basis, residues)` -- D9: DecQ_p(r_0, ..., r_k) = Dec_p(r_0, ..., r_k) mod q. Asserts `basis.capacity() >= q` at runtime.
- `encode(basis, c)` -- inverse direction: lifts a Z_q coefficient to Clockwork residues.
- `verify_roundtrip(basis, c)` -- CT-05 test obligation: encode then decode must return the original.

### garner (k_eliminate) -- Formal Spec D5-D6

**File**: `garner.rs`

The K-Elimination 2-gear step and full Garner mixed-radix decomposition.

- `k_eliminate(r, s, m, a, m_inv_mod_a)` -- D6: compute k = (s - r) * m^(-1) mod a. T2 guarantees k in {0, ..., a-1} and X = r + k*m (mod m*a).
- `k_eliminate_ct(...)` -- constant-time version for secret-dependent paths. Uses uniform execution time regardless of input values (T16 requirement). All subtraction is done via `(s + a - r_mod_a) % a` to avoid branching.
- `garner_decompose(residues, moduli)` -- D5: full iterative Garner algorithm producing `GarnerDigits`. Each step is a K-Elimination.
- `garner_decompose_ct(...)` -- constant-time version of the full decomposition.
- `GarnerDigits::reconstruct()` -- recover X = d_0 + d_1*p_0 + d_2*p_0*p_1 + ...

Tested exhaustively in CT-02 (K-Elimination vs naive CRT decode) and with 1M random samples for the large-modulus case.

### gearstack (GearStack) -- composite working type

**File**: `gearstack.rs`

The primary working type in Clockwork: holds a value's RNS residues, a reference to its basis, and a bound tracker entry. Every arithmetic operation returns a new `GearStack` with an updated bound derived from the operand bounds (never from the actual value).

- `GearStack::from_value(value, bound_bits, basis)` -- create from unsigned value.
- `GearStack::from_signed(value, bound_bits, basis)` -- create from signed value.
- `add(&self, other)` -- lane-wise addition with D7 bound update.
- `sub(&self, other)` -- lane-wise subtraction.
- `mul(&self, other)` -- lane-wise multiplication.
- `verify_bound()` -- DEBUG/TEST only: reconstructs the full value and checks it against the claimed bound (CT-03).
- `needs_promotion(delta_bits)` -- checks if the basis has capacity for an operation that increases the bound.
- `dot_product(weights, values)` -- dot product with tracked bounds.

Default guard margin is 8 bits (`DEFAULT_GUARD_BITS`).

### gro (GroGate) -- Formal Spec D13-D16

**File**: `gro.rs`

Golden Ratio Oscillator pair timing model for side-channel protection. This solves the B5 k-leakage problem: timing isolation ensures K-Elimination reveals nothing through execution time.

- `GroGate::new(params)` -- construct from explicit phase increments. Validates N_acc in [8, 63], nonzero increments, and reasonable window width.
- `GroGate::golden_ratio(n_acc, delta_phi_a, window_width)` -- construct with phi-approximating phase increments using Fibonacci ratios (F_86/F_85). Adjusts to ensure odd phase difference for maximal period (T9).
- `is_window(t)` -- D14: is time step t inside a coincidence window? True when `|phase_A(t) - phase_B(t)| < window_width`.
- `coincidence_period()` -- T9: `T_coinc = 2^N_acc / gcd(delta_phi_A - delta_phi_B, 2^N_acc)`. When the difference is odd, the period is maximal: `2^N_acc`.
- `next_window(from, max_search)` -- find the next coincidence window.
- `count_windows_in_range(from, to)` -- for equidistribution testing (GT-02).

All phase computation uses wrapping integer multiplication and bit masking. The golden ratio is approximated via `F(86)/F(85)` with integer division only (Axiom A3: DDS frequencies are rational).

### integrity (TripleRedundant) -- Formal Spec D22-D23

**File**: `integrity.rs`

Triple-redundant storage with CRC32 verification for tier state and GearStack metadata.

- `TripleRedundant::new(value)` -- D22: stores three copies and a CRC32 checksum.
- `read()` -- D23 MajVote algorithm. Returns:
  - `VoteResult::AllAgree(value)` -- no corruption.
  - `VoteResult::Recovered { value, corrupted_copy }` -- single-location corruption detected and corrected (T14).
  - `VoteResult::Failed` -- two or more copies corrupted, fail-closed (T15).
- `write(value)` -- overwrites all copies and recomputes CRC.
- `repair()` -- restores triple redundancy after single-copy corruption.

The `TierState` struct (num_gears, bound_bits, active_moduli) implements `AsBytes` for CRC computation.

Test IT-03 verifies CRC collision resistance: over 1M random double-corruption patterns, detection rate exceeds 99.99%.

### key_lifecycle -- Formal Spec D18-D21

**File**: `key_lifecycle.rs`

FHE secret key lifecycle management with additive sharing and scheduled resharing.

- `KeySharePair::split(s, r, q)` -- D18: s1 = r, s2 = (s - r) mod q. T12 guarantees s1 is uniform in Z_q, independent of s.
- `KeySharePair::reconstruct()` -- recover s = s1 + s2 mod q. WARNING: puts full key in memory; must be zeroed after use.
- `KeySharePair::reshare(r)` -- D19: s1' = s1 + r, s2' = s2 - r (mod q). T11 guarantees s1' + s2' = s.
- `KeyLifecycle` state machine (D20): `Keygen -> Split -> Active -> Zeroed`, with reshare cycles during Active.
  - `initialize(s, r)` -- splits key and transitions to Active.
  - `record_operation()` -- returns true when reshare interval reached (public, schedule-based per INV-7).
  - `reshare(r)` -- reshares with fresh randomness; old shares zeroed by Drop.
  - `destroy()` -- zeroes all material, transitions to Zeroed (terminal).

Memory zeroing (Axiom A4) uses `zeroize::Zeroize` with compiler barriers to prevent dead-store elimination. `KeySharePair` implements `Drop` to auto-zero on scope exit.

This module is the ONLY place in `clockwork-core` that uses `unsafe` -- specifically through the `zeroize` crate's volatile zeroing mechanism.

---

## Zero float guarantee

The crate enforces zero floating-point arithmetic at the compiler level:

```rust
#![deny(clippy::float_arithmetic)]
#![deny(clippy::float_cmp)]
```

Combined with CI `grep` checks, this guarantees no `f32` or `f64` values exist anywhere in the crate.

---

## Safety

The crate declares `#![deny(unsafe_code)]` at the crate root. The only exception is `key_lifecycle.rs`, which uses the `zeroize` crate's volatile zeroing mechanism (required by Axiom A4 for guaranteed memory clearing). All other code is safe Rust.

---

## Relationship to nine65::arithmetic

| Concern | clockwork-core | nine65::arithmetic |
|---------|---------------|-------------------|
| Purpose | Formal spec compliance, reference implementation | Performance-optimized FHE arithmetic |
| K-Elimination | `garner::k_eliminate` (spec-aligned, both standard and CT) | `k_elimination::KElimination` (production, with DualRNS support) |
| NTT | Not included | `NTTEngine`, `NTTEngineFFT` |
| Montgomery | Not included | `PersistentMontgomery`, `MontgomeryContext` |
| Bound tracking | `Bound` (formal D7 rules) | Noise budget (millibits) via `NoiseBudget` |
| Integrity | `TripleRedundant` (CRC32 + MajVote) | Not duplicated -- uses clockwork-core's types |

The `clockwork` feature flag in `nine65` enables integration: GRO timing gates, bound tracking, key lifecycle, and integrity checks from this crate.

---

## Running tests

```bash
cargo test -p clockwork-core --release
```

All 46 tests should pass. The test suite covers:

- **CT-01**: Exhaustive CRT encode/decode roundtrip (all X in [0, M_k) for small bases, 1M random samples for larger bases).
- **CT-02**: K-Elimination vs naive CRT decode (exhaustive for small M, 1M random for large).
- **CT-03**: Bound tracker soundness verification (100K samples, worst-case addition and multiplication).
- **CT-04**: Promotion/demotion roundtrip (100K samples).
- **CT-05**: DecQ roundtrip for all c in [0, q) across 5 different basis choices.
- **CT-06**: Lane-wise add/mul matches ring operations (100K samples each).
- **GT-01**: GRO coincidence period = 2^N_acc for odd phase differences (100 random parameter sets).
- **GT-02**: GRO window equidistribution across 16 bins over one full period.
- **IT-01 through IT-04**: Triple-redundant corruption detection, recovery, and CRC collision resistance.
- **KT-01 through KT-04**: Key share correctness, memory zeroing, lifecycle state machine, and share distribution uniformity.

---

## Where to go next

- [Noise Budget](noise-budget) -- how nine65's runtime noise tracking builds on these formal bounds.
- [Bootstrap](bootstrap) -- the three bootstrap paths that use K-Elimination from this spec.
- [Formal Proofs](formal-proofs) -- the Coq and Lean4 proofs that verify these algorithms.
