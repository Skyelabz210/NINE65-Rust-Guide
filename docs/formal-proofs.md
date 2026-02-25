---
layout: default
title: Formal Proofs
---

# Formal Proofs

[Home](index)

## What Formal Verification Means

The Coq and Lean4 proofs are machine-checked mathematical proofs that certain properties of the system hold for ALL inputs, not just the ones covered by tests. A test says "this worked for these 1000 inputs." A formal proof says "this works for every possible input, mathematically guaranteed."

## Coq Proofs (14)

Location: `proofs/coq/`

| Proof | What it proves |
|-------|---------------|
| K-Elimination | For coprime alpha, beta, and V < alpha*beta, reconstruction is exact |
| GSO-FHE | Depth bounds for ciphertext operations |
| CRT Shadow Entropy | Minimum entropy guarantee for shadow harvester |
| Order Finding | Correctness of modular order computation |
| MQ-ReLU | Modular quadratic ReLU activation correctness |
| Integer Softmax | Integer softmax produces valid probability distribution |
| Montgomery | montgomery_reduce(a * R) == a mod q |
| Mobius | Mobius function computation correctness |
| Cyclotomic Phase | Phase computation in cyclotomic rings |
| Pade Engine | Pade approximation engine correctness |
| Exact Coefficient | Coefficient extraction is exact |
| State Compression | State compression preserves information |
| Side-Channel Resistance | Constant-time operation guarantees |
| Encrypted Quantum | Encrypted quantum state operations |

### Running Coq proofs

Requires Coq 8.18+:

```bash
cd proofs/coq
coqc *.v
```

## Lean4 Proofs (4)

Location: `lean4/KElimination/`

| Proof | What it proves |
|-------|---------------|
| K-Elimination | Core K-Elimination theorem (same as Coq, different proof assistant) |
| Core Definitions | Foundational type definitions and properties |
| Shadow Entropy | Shadow entropy security predicate |
| Modular Arithmetic | Basic modular arithmetic properties used throughout |

### Running Lean4 proofs

Requires Lean 4.x with Mathlib:

```bash
cd lean4/KElimination
lake build
```

## Connection to Tests

The `formal_spec_linkage` module in the extreme test harness generates test vectors from the formal predicates and verifies the Rust implementation matches:

```bash
cargo test -p nine65-extreme-tests --features extreme-tests --release -- formal_spec_linkage --nocapture
```

This bridges the gap between "the math is proven correct" and "the code implements the math correctly."
