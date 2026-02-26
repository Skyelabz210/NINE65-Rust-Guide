---
layout: default
title: Circuit Compiler
parent: How It Works
nav_order: 13
---

# Circuit Compiler

[Home](index)

The circuit compiler performs static noise analysis on FHE circuits and selects parameters that guarantee bootstrap-free execution. Given a circuit DAG, it computes the maximum noise accumulation across all paths and determines the modulus chain needed to contain that noise without ever bootstrapping.

**Source**: `crates/nine65/src/compiler.rs`

---

## Float exception notice

This module is the ONLY place in the entire NINE65 workspace that allows floating-point arithmetic:

```rust
#![allow(clippy::float_arithmetic)]
```

Floats are permitted here because the circuit compiler is a **compile-time static analysis tool**, not a runtime computation engine. It estimates noise growth in bits using `f64` for convenience during parameter selection. The actual FHE operations at runtime remain 100% integer-only. No ciphertext, key material, or encrypted data ever touches a float.

---

## Circuit representation

### OpType enum

The compiler models FHE circuits as directed acyclic graphs (DAGs) where each node is one of:

| OpType | Description |
|--------|-------------|
| `Input` | Circuit input (encrypted value entering the computation). |
| `Output` | Circuit output (encrypted result leaving the computation). |
| `Add` | Homomorphic addition of two ciphertexts. |
| `Multiply` | Homomorphic multiplication of two ciphertexts. |
| `Rescale` | Modulus switching to reduce noise (drops one prime from the modulus chain). |
| `Relinearize` | Key-switching to reduce ciphertext degree after multiplication. |
| `Rotate` | Slot rotation (Galois automorphism). |

### CircuitNode

Each node in the DAG tracks:

```rust
pub struct CircuitNode {
    pub id: usize,
    pub op_type: OpType,
    pub inputs: Vec<usize>,          // IDs of input nodes
    pub depth: usize,                // Total depth from inputs
    pub multiplicative_depth: usize, // Multiplicative levels from inputs
}
```

The `depth` is the length of the longest path from any `Input` node. The `multiplicative_depth` only counts `Multiply` nodes along that path. This distinction matters because additions and rotations contribute less noise than multiplications.

### Circuit

The `Circuit` struct holds the complete DAG:

```rust
pub struct Circuit {
    pub nodes: Vec<CircuitNode>,
    pub input_nodes: Vec<usize>,
    pub output_nodes: Vec<usize>,
    pub max_depth: usize,
    pub max_multiplicative_depth: usize,
}
```

Building a circuit:

```rust
use nine65::compiler::{Circuit, OpType};

let mut circuit = Circuit::new();

let x = circuit.add_node(OpType::Input, vec![]);
let x2 = circuit.add_node(OpType::Multiply, vec![x, x]);
let x2_relin = circuit.add_node(OpType::Relinearize, vec![x2]);
let x2_rescale = circuit.add_node(OpType::Rescale, vec![x2_relin]);
let result = circuit.add_node(OpType::Add, vec![x, x2_rescale]);
circuit.add_node(OpType::Output, vec![result]);
```

When `add_node` is called, depth and multiplicative depth are computed automatically from the input nodes. `operation_counts()` returns a `HashMap<OpType, usize>` summarizing the circuit.

---

## Static noise analysis

### NoiseModel

The noise model assigns a bit-cost to each operation:

```rust
pub struct NoiseModel {
    pub add_noise_bits: f64,           // Default: 2.0
    pub mul_noise_bits: f64,           // Default: 25.0 (log2(t) + overhead)
    pub relin_noise_bits: f64,         // Default: 15.0
    pub rescale_reduction_bits: f64,   // Default: 60.0 (typical scale bits)
    pub rotate_noise_bits: f64,        // Default: 5.0
    pub safety_factor: f64,            // Default: 1.3 (30% margin)
}
```

`NoiseModel::conservative()` returns the default model. The `noise_for_op(op_type)` method returns the noise contribution for a given operation type, multiplied by the safety factor. Rescale returns a negative value (it reduces noise).

### NoiseAnalyzer

The analyzer performs a topological traversal of the circuit DAG:

1. For each node, start with the maximum noise from its input nodes (or 3.2 bits for nodes with no inputs -- initial encryption noise).
2. Add the noise contribution of the current operation.
3. Clamp to non-negative.
4. Track the maximum noise across all nodes and the noise at output nodes.

The result is a `NoiseAnalysisResult`:

```rust
pub struct NoiseAnalysisResult {
    pub max_noise_bits: f64,
    pub output_noise_bits: f64,
    pub per_node_noise: HashMap<usize, f64>,
}
```

---

## Parameter selection

### ParameterSelector

Given a circuit's noise analysis, the selector determines FHE parameters:

```rust
let selector = ParameterSelector::new(128, 16); // 128-bit security, 16-bit plaintext
let params = selector.select_for_circuit(&circuit, &noise_result);
```

The required total modulus bits are computed as:

```
required_bits = max_noise + plaintext_bits + security_level + 20 (safety)
```

From this, the selector:
1. Computes the number of 60-bit primes needed: `ceil(required_bits / 60)`.
2. Selects primes from a precomputed list of NTT-friendly 60-bit primes.
3. Sets the polynomial degree based on security level: N=8192 for 128-bit, N=16384 for 192-bit, N=32768 for 256-bit.

### FHEParameters

The output:

```rust
pub struct FHEParameters {
    pub poly_degree: usize,
    pub modulus_chain: Vec<u64>,
    pub plaintext_modulus: u64,
    pub security_bits: usize,
    pub total_modulus_bits: usize,
    pub multiplicative_depth_supported: usize,
    pub bootstrap_free: bool,
}
```

The `summary()` method returns a human-readable string of all parameters.

---

## The compilation pipeline

`BootstrapFreeFHECompiler` ties everything together:

```rust
let compiler = BootstrapFreeFHECompiler::new(128, 16);
let result = compiler.compile(&circuit);
```

The pipeline runs four steps:

1. **Circuit analysis** -- count nodes, compute depths, enumerate operations.
2. **Static noise analysis** -- traverse the DAG and compute per-node noise.
3. **Parameter selection** -- choose modulus chain and polynomial degree.
4. **Bootstrap-free verification** -- check that `available_budget >= max_noise`. The budget is `total_modulus_bits - plaintext_bits - security_level`.

The `CompilationResult` contains the circuit, parameters, noise analysis, and a `bootstrap_free_guaranteed` boolean.

### Speedup estimation

`estimate_speedup(&result)` computes the theoretical speedup over traditional FHE:

- Traditional FHE bootstraps approximately every 15 multiplicative levels.
- Each bootstrap costs roughly 5000x a regular operation.
- The ratio `(nodes + bootstraps * 5000) / nodes` gives the speedup factor.

For a depth-50 circuit, this yields roughly a 17x speedup estimate.

---

## Example circuits

The module includes two factory functions:

### example_polynomial_circuit()

Computes `x + x^2 + x^3 + x^4`:

```
Input(x)
  -> Mul(x, x) -> Relin -> Rescale = x^2
  -> Mul(x^2, x) -> Relin -> Rescale = x^3
  -> Mul(x^2, x^2) -> Relin -> Rescale = x^4
  -> Add(x, x^2) -> Add(_, x^3) -> Add(_, x^4)
  -> Output
```

Multiplicative depth: 2 (two sequential multiplications for x^4).

### example_deep_circuit(depth)

A chain of `depth` sequential squarings:

```
Input -> [Mul -> Relin -> Rescale] x depth -> Output
```

Multiplicative depth equals the `depth` parameter. Used for depth-50 verification.

---

## How to use

Define a circuit, compile it, and use the recommended parameters:

```rust
use nine65::compiler::*;

// Build your circuit
let mut circuit = Circuit::new();
let x = circuit.add_node(OpType::Input, vec![]);
let y = circuit.add_node(OpType::Input, vec![]);
let sum = circuit.add_node(OpType::Add, vec![x, y]);
let prod = circuit.add_node(OpType::Multiply, vec![sum, x]);
let prod_relin = circuit.add_node(OpType::Relinearize, vec![prod]);
let prod_rescale = circuit.add_node(OpType::Rescale, vec![prod_relin]);
circuit.add_node(OpType::Output, vec![prod_rescale]);

// Compile
let compiler = BootstrapFreeFHECompiler::new(128, 16);
let result = compiler.compile(&circuit);

// Check guarantee
assert!(result.bootstrap_free_guaranteed);
println!("{}", result.parameters.summary());
println!("Speedup: {:.1}x", compiler.estimate_speedup(&result));
```

The compiler prints detailed output at each step. The `parameters` field contains the recommended `poly_degree`, `modulus_chain`, and `plaintext_modulus` that you can feed into `FHEConfig::custom()` for actual FHE operations.

---

## Tests

The compiler module includes four tests:

- `test_polynomial_circuit_compilation` -- compiles `x + x^2 + x^3 + x^4`, asserts bootstrap-free.
- `test_deep_circuit_compilation` -- compiles circuits at depths 5, 10, and 20, asserts all are bootstrap-free.
- `test_parameter_scaling` -- compiles a depth-10 circuit at security levels 128, 192, and 256, showing how parameters scale.
- `test_depth_50_bootstrap_free` -- the verification of `GSOFHE.v:depth_50_achievable`: a 50-level multiplicative circuit compiled bootstrap-free with total modulus bits under 2000.

Run them:

```bash
cargo test -p nine65 --lib --release -- compiler::tests --nocapture
```

---

## Where to go next

- [Noise Budget](noise-budget) -- how runtime noise tracking works after the compiler selects parameters.
- [Bootstrap](bootstrap) -- the three bootstrap paths (for when circuits exceed static analysis bounds).
- [Architecture](architecture) -- how the compiler fits into the overall system.
