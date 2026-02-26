---
layout: default
title: Neural Operations
parent: Using the System
nav_order: 5
---

# Neural Operations

[Home](index)

Integer-only neural network primitives for encrypted inference. Every activation function, every arithmetic operation -- zero floating-point.

Source files: `ops/neural.rs`, `arithmetic/mq_relu.rs`, `arithmetic/pade_engine.rs`, `arithmetic/integer_softmax.rs`, `arithmetic/cyclotomic_phase.rs`, `arithmetic/mobius_int.rs`.

---

## FHENeuralEvaluator

The unified interface for neural network operations on encrypted data.

```rust
use nine65::ops::neural::{FHENeuralEvaluator, ActivationType};

let evaluator = FHENeuralEvaluator::new(modulus, plaintext_mod);

// Or with custom softmax scale
let evaluator = FHENeuralEvaluator::with_softmax_scale(modulus, plaintext_mod, 1_000_000_000_000);
```

### ActivationType

```rust
pub enum ActivationType {
    None,       // Linear pass-through
    ReLU,       // max(0, x) via MQ-ReLU O(1) threshold
    LeakyReLU,  // Configurable leak coefficient
    Sigmoid,    // 1/(1+exp(-x)) via Pade [4/4]
    Tanh,       // (exp(x)-exp(-x))/(exp(x)+exp(-x)) via Pade
    Softmax,    // Exact sum guarantee
    GELU,       // x * sigmoid(1.702x) approximation
}
```

### Dense Layer Forward Pass

```rust
let output = evaluator.dense_forward(
    &input,      // &[MobiusInt]
    &weights,    // &[Vec<MobiusInt>]
    &bias,       // &[MobiusInt]
    ActivationType::ReLU,
);
```

Computes `output[i] = activation(sum_j(weights[i][j] * input[j]) + bias[i])` using MobiusInt arithmetic.

---

## MQ-ReLU: O(1) Sign Detection

Defined in `arithmetic/mq_relu.rs`. Replaces traditional comparison circuits with a simple threshold check: if `value > q/2`, it is "negative."

**Coq-verified**: `MQReLU.v` -- `sign_detection_correct`, `mq_relu_correct`, `speedup_is_2000x`.

### Sign Enum

```rust
pub enum Sign {
    Positive,  // value in [1, q/2)
    Negative,  // value in [q/2, q)
    Zero,      // value == 0
}
```

### MQReLU Type

```rust
use nine65::arithmetic::MQReLU;

let relu = MQReLU::new(modulus);

// Sign detection (constant-time via subtle crate)
let sign = relu.detect_sign(value); // -> Sign::Positive, Negative, or Zero

// ReLU: max(0, x) -- returns value if positive, 0 otherwise
let activated = relu.apply_scalar(value);

// Apply to entire polynomial
let activated_poly = relu.apply_polynomial(&coeffs);

// Leaky ReLU: positive -> value, negative -> |x| * leak_num / leak_den
let leaky = relu.leaky_relu_scalar(value, 1, 10); // leak = 0.1
let leaky_poly = relu.leaky_relu_polynomial(&coeffs, 1, 10);

// Custom threshold
let custom = MQReLU::with_threshold(modulus, custom_threshold);
```

**Performance**: ~20ns per coefficient. The sign detection uses `subtle::ConstantTimeLess` and `subtle::ConstantTimeEq` for constant-time operation on the sensitive computation path.

---

## PadeEngine: Transcendentals via Rational Approximation

Defined in `arithmetic/pade_engine.rs`. Computes exp, sin, cos, ln, sigmoid, tanh using Pade [4/4] rational functions with integer coefficients.

**Coq-verified**: `PadeEngine.v`.

### PADE_SCALE Constant

```rust
pub const PADE_SCALE: i128 = 1_000_000_000; // 10^9 represents 1.0
```

All inputs and outputs are scaled integers. A value `x` in the real domain is represented as `x * PADE_SCALE`.

### Core Functions

```rust
use nine65::arithmetic::PadeEngine;

let pade = PadeEngine::default(); // uses PADE_SCALE

// Exponential: input x (scaled), output exp(x/SCALE) * SCALE
let exp_val = pade.exp_integer(500_000_000); // exp(0.5) * 10^9

// Sine and Cosine
let sin_val = pade.sin_integer(x);
let cos_val = pade.cos_integer(x);

// Natural logarithm
let ln_val = pade.ln_integer(x); // x must be > 0

// Sigmoid: 1 / (1 + exp(-x))
let sig = pade.sigmoid_integer(x);

// Tanh: (exp(x) - exp(-x)) / (exp(x) + exp(-x))
let th = pade.tanh_integer(x);
```

When the `exact_rational` feature is enabled, the engine delegates to the `exact_transcendentals` backend for higher accuracy. Otherwise, it uses the built-in Pade [4/4] approximation.

### Pade [4/4] Coefficients for exp(x)

```
P(x) = 1680 + 840x + 180x^2 + 20x^3 + x^4
Q(x) = 1680 - 840x + 180x^2 - 20x^3 + x^4
exp(x) ~ P(x) / Q(x)
```

Evaluation uses Horner's method (integer only). Accuracy: error < 10^-8 for |x| < 1 in the scaled domain.

**Performance**: ~200ns per evaluation, zero drift, fully reproducible.

### Via FHENeuralEvaluator

```rust
let sigmoid = evaluator.sigmoid(x); // i128 -> i128
let tanh = evaluator.tanh(x);
let exp = evaluator.exp(x);
let gelu = evaluator.gelu(x);       // x * sigmoid(1.702 * x)
```

---

## IntegerSoftmax: Exact Sum Guarantee

Defined in `arithmetic/integer_softmax.rs`. Computes softmax using Pade exp() and guarantees `sum(output) == SOFTMAX_SCALE` exactly.

**Coq-verified**: `IntegerSoftmax.v`.

### SOFTMAX_SCALE Constant

```rust
pub const SOFTMAX_SCALE: u128 = 1_000_000_000_000; // 10^12
```

The sum of all softmax outputs equals this value exactly -- not approximately.

### Usage

```rust
use nine65::arithmetic::IntegerSoftmax;

let softmax = IntegerSoftmax::new();

// Input: logits as scaled integers
let logits = vec![1000i128, 2000, 3000];
let probs = softmax.compute(&logits);

// probs[0] + probs[1] + probs[2] == SOFTMAX_SCALE exactly
let sum: u128 = probs.iter().sum();
assert_eq!(sum, 1_000_000_000_000);

// Custom scale
let softmax = IntegerSoftmax::with_scale(1_000_000);
```

### Algorithm

1. Find max logit (integer comparison)
2. Shift inputs for numerical stability: `shifted[i] = logits[i] - max`
3. Compute `exp()` via Pade [4/4] for each shifted value
4. Sum all exp values
5. Divide each by total
6. **Adjust rounding** to guarantee exact sum (the key innovation)

The adjustment step distributes any rounding remainder across the largest output values, ensuring the sum is always exactly `SOFTMAX_SCALE`.

### Via FHENeuralEvaluator

```rust
let probs = evaluator.softmax(&logits); // Vec<u128>
```

---

## CyclotomicPhase: Ring-Native Trigonometry

Defined in `arithmetic/cyclotomic_phase.rs`. The ring `R_q[X]/(X^N + 1)` has native trigonometric properties because `X^N = -1`, so `X^k` is a phase rotation by `k * (pi/N)`.

**Coq-verified**: `CyclotomicPhase.v`.

No polynomial approximation needed -- sine and cosine are native to the ring structure.

### Types

```rust
use nine65::arithmetic::cyclotomic_phase::{CyclotomicRing, CyclotomicPolynomial};

// Create ring with primitive root detection
let ring = CyclotomicRing::new(n, q);
// Fields: ring.n, ring.q, ring.psi (primitive root), ring.psi_inv

// Create polynomial in the ring
let poly = CyclotomicPolynomial::new(coeffs, ring.clone());
let zero = CyclotomicPolynomial::zero(ring.clone());

// Create a pure phase element X^k
let phase = CyclotomicPolynomial::phase(k, ring.clone());
```

- **Odd coefficients** of a polynomial represent "sine" components
- **Even coefficients** represent "cosine" components

**Performance**: ~50ns for phase extraction (vs ~3ms for traditional polynomial approximation of trig functions).

---

## MobiusInt: Signed Arithmetic Without Threshold Failures

Defined in `arithmetic/mobius_int.rs`. Separates magnitude from sign to avoid the classic M/2 threshold problem in modular signed arithmetic.

**Coq-verified**: `MobiusInt.v`.

### The Problem

Traditional approach: if `residue > M/2`, treat as negative. After multiple chained operations, the threshold check gives wrong answers because intermediate values can wrap around unpredictably.

### The Solution: Polarity Separation

```rust
use nine65::arithmetic::{MobiusInt, Polarity};

pub struct MobiusInt {
    pub residue: u64,     // magnitude (always non-negative)
    pub polarity: Polarity, // Plus or Minus
}

pub enum Polarity {
    Plus,
    Minus,
}
```

### Arithmetic

```rust
// Construction
let a = MobiusInt::from_i64(-42);
let b = MobiusInt::from_i64(7);
let zero = MobiusInt::zero();
let one = MobiusInt::one();

// Arithmetic (polarity propagates correctly)
let sum = a.add(&b);      // -42 + 7 = -35
let prod = a.mul(&b);     // -42 * 7 = -294 (polarity: Minus XOR Plus = Minus)
let neg = a.neg();         // 42 (polarity flipped)

// Polarity operations
let p = Polarity::Plus.xor(Polarity::Minus); // Minus
let q = Polarity::Minus.flip();               // Plus
```

### Via FHENeuralEvaluator

```rust
// Convert residue to signed representation
let signed = evaluator.to_signed(residue);

// Convert back
let residue = evaluator.from_signed(&signed);

// Batch conversion
let signed_poly = evaluator.poly_to_signed(&coeffs);
let residue_poly = evaluator.poly_from_signed(&signed_poly);
```

**Performance**: ~15ns per operation, exact, no threshold errors even after arbitrary chaining.

---

## Performance Summary

| Component | Time | vs Standard FHE |
|-----------|------|----------------|
| MQ-ReLU sign detection | ~20ns | 2000x faster than comparison circuit (~2ms) |
| Pade exp/sigmoid/tanh | ~200ns | 250x faster than polynomial approximation (~50ms) |
| CyclotomicPhase sin/cos | ~50ns | 60x faster than Taylor approximation (~3ms) |
| IntegerSoftmax per element | ~200ns | Exact sum (standard FHE cannot guarantee sum = 1.0) |
| MobiusInt arithmetic | ~15ns | 100% accuracy (vs 0% under chained M/2 threshold) |

Combined speedup for neural inference: 1,000 to 100,000x faster than standard FHE polynomial approximation approaches.

---

## Where to go next

- [Batch and Galois](batch-and-galois) -- packing multiple values for SIMD-like operations
- [MANA Accelerator](mana) -- parallel stream processing for neural workloads
- [Cookbook](cookbook) -- complete FHE neural network example
