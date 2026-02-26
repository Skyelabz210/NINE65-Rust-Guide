---
layout: default
title: Benchmarks
parent: Tools & Testing
nav_order: 2
---

# Benchmarks

[Home](index)

NINE65 provides three benchmarking approaches: a standalone binary that outputs structured JSON, Criterion micro-benchmarks for per-operation latency and throughput, and in-tree benchmark tests that run under `cargo test`.

---

## nine65_bench binary

The `nine65_bench` binary runs real FHE operations and outputs structured JSON suitable for dashboards, speedometers, and the hackfate.us demo page.

### Build

```bash
cargo build --release -p nine65 --bin nine65_bench --features serde
```

### Run

```bash
nine65_bench --config secure_128 --max-depth 80 --output bench.json
```

### Arguments

| Flag | Values | Default | Description |
|------|--------|---------|-------------|
| `--config` | `secure_128`, `secure_128_deep`, `secure_192`, `secure_256`, `standard_128`, `high_192` | `secure_128` | Security configuration to benchmark. `standard_128` maps to `secure_128`; `high_192` maps to `secure_192`. |
| `--max-depth` | any positive integer | `80` | Maximum depth for the mixed-operation chain test. |
| `--output` / `-o` | file path | stdout | Write JSON to a file instead of stdout. |
| `--a` | `u64` value | `8` | First operand for the initial `a * b` computation. |
| `--b` | `u64` value | `8` | Second operand for the initial `a * b` computation. |
| `-h` / `--help` | | | Print usage and exit. |

### What it measures

1. **Keygen timing** -- single key generation call, reported in microseconds.
2. **Single-operation latency** -- each operation averaged over 100 iterations: encrypt, decrypt, add (ct+ct), sub (ct-ct), negate, add_plain, mul_plain, mul (ct*ct).
3. **Depth chain** -- a sequence of `mul_plain`, `add_plain`, and `sub_plain` operations cycling through a fixed 20-operation pattern. At each step the harness tracks noise budget percentage, elapsed time, and whether a Clockwork Bootstrap refresh was simulated. After the chain completes, it decrypts the result and verifies correctness against a parallel plaintext computation.
4. **Scale tests** -- four workload simulations up to configurable depth:
   - `deep_arithmetic` (80 ops): alternating `mul_plain` and `add_plain`.
   - `statistical_pipeline` (60 ops): accumulation via `add_plain`.
   - `neural_network` (50 ops): alternating `mul_plain` (dense layer proxy) and `add_plain` (bias proxy).
   - `polynomial_eval` (128 ops): Horner-method style `mul_plain` + `add_plain`.
5. **Speedometer summary** -- derived aggregates: average ops/sec, average latency, max depth achieved, depth/sec, minimum noise budget, total refreshes.

### JSON output structure

The JSON contains top-level keys: `metadata`, `keygen_us`, `operations`, `depth_chain`, `scale_tests`, and `speedometer_summary`. The `metadata` block includes the NINE65 crate version, config parameters (n, q, t, eta, security_bits), seed, and a timestamp.

Changes to the NINE65 codebase ARE reflected when you rebuild and rerun the binary. It imports directly from `nine65::prelude::*` and uses the same `SecureConfig`, `KeySet`, `BFVEncoder`, `BFVEncryptor`, `BFVDecryptor`, `BFVEvaluator`, `NTTEngine`, `ShadowHarvester`, and `NoiseBudget` that the library exposes.

---

## Criterion bench files

All Criterion benchmarks live in `crates/nine65/benches/`. Run them with:

```bash
cargo bench -p nine65
```

To run a specific bench file:

```bash
cargo bench -p nine65 --bench timing
```

### timing.rs -- per-operation latency

Measures constant-time primitive latencies:

- **barrett_ct**: Barrett reduction (small and large inputs) and Barrett multiply.
- **k_elimination_ct**: `extract_k` across small, large, and edge-case inputs. Also benchmarks `mul_mod_u128_ct` and `sub_mod_u128_ct`.
- **k_elimination_divider**: `ExactDivider` operations -- `reconstruct_exact`, `exact_divide`, `divmod`, and `scale_and_round` (the BFV-like rescaling path).
- **ntt_ct**: Forward NTT, inverse NTT, and polynomial multiply at N=256 with small and large coefficient inputs.
- **ntt_fft**: FFT-based NTT multiply at N=1024 and N=4096 (requires `ntt_fft` feature, enabled by default).
- **rns_kelim_rescale**: K-Elimination rescale on a `secure_128` ciphertext (the inner loop of BFV multiplication).

### throughput.rs -- operations per second

Measures batch and parallel throughput. Requires `--features parallel`.

```bash
cargo bench -p nine65 --bench throughput --features parallel
```

- **batch_encoding**: `BatchEncoder` encode/decode at batch sizes 64, 256, 512, 1024.
- **parallel_encryption**: Sequential vs `ParallelEncryptor` at batch sizes 10, 50, 100, 500.
- **parallel_decryption**: Sequential vs `ParallelDecryptor` at the same batch sizes.
- **roundtrip_throughput**: Full encrypt-then-decrypt roundtrip at batch sizes 100, 500, 1000.

All throughput benchmarks report `Throughput::Elements` so Criterion computes elements/sec automatically.

### fhe_scaling.rs -- O(N log N) FFT scaling

Demonstrates that FHE operation time scales with ring dimension N as expected:

- **homo_mul_scaling**: Homomorphic multiply across `secure_128` (N=2048), `secure_128_deep` (N=4096), `secure_192` (N=4096), and `secure_256` (N=8192). All configs use production 128-bit+ security.
- **ntt_scaling**: Forward NTT and NTT roundtrip at N=1024, 2048, 4096, 8192.
- **encrypt_decrypt_scaling**: Encrypt and decrypt latency at N=2048, 4096, 8192.

### adaptive_rayon.rs -- AdaptiveFHEContext vs static ParallelEncryptor

Requires `--features parallel`:

```bash
cargo bench -p nine65 --bench adaptive_rayon --features parallel
```

- **adaptive_vs_static_encrypt**: Entropy-monitored `AdaptiveFHEContext` versus fixed-pool `ParallelEncryptor` at batch sizes 5, 20, 50, 100.
- **adaptive_vs_static_decrypt**: Same comparison for decryption at batch sizes 10, 50, 100.
- **entropy_overhead**: Cost of `measure_entropy_from_ciphertext`, `adapt_threading`, and `is_high_entropy` calls on the `ShadowEntropyMonitor`.
- **thread_pool_creation**: Rayon `ThreadPoolBuilder::new().build()` latency for 1, 2, 4, and 8 threads.
- **adaptive_add**: Adaptive homomorphic add throughput at batch sizes 5, 15, 50.

### threading_comparison.rs -- thread strategy sweep

Requires `--features parallel`:

```bash
cargo bench -p nine65 --bench threading_comparison --features parallel
```

Compares three encryption strategies at batch sizes 5, 20, 50, 100:

1. **sequential** -- single-threaded, creating NTT/encoder per message.
2. **generic_rayon** -- `par_iter` with `map_init` for per-thread NTT/encoder caching.
3. **adaptive_system** -- `AdaptiveFHEContext::adaptive_encrypt` with entropy-driven thread adaptation.

### nine65_vs_seal_comparison.rs -- native Rust benchmarks

This file benchmarks NINE65's native Rust implementations of operations that have equivalents in SEAL:

- **Persistent Montgomery**: enter, mul, exit operations.
- **Order Finding**: `multiplicative_order` and `factor_semiprime`.
- **Polynomial Operations**: Horner evaluation at sizes 100, 1000, 4096.
- **Sign Detection (MQ-ReLU)**: `MobiusInt` sign detection and `MQ-ReLU` scalar application.
- **Quantum Operations**: Grover optimal iteration computation (integer approximation).
- **ML Operations**: `IntegerSoftmax` at sizes 10, 100, 1000 and `MQ-ReLU` polynomial on 4096-element vectors.
- **Pade Transcendentals**: `PadeEngine` exp, sin, sigmoid on scaled integer inputs.
- **Cyclotomic Phase**: rotation and sine/cosine extraction on a 4096-degree ring.
- **FHE_DualRNS_K-Elimination**: Encrypt, decrypt, add, and `mul_k_elim` (ciphertext multiplication with K-Elimination rescale) using `RNSFHEContext` with `secure_128`.
- **FHE_KeyGen**: `generate_keys_dual` timing.

---

## comprehensive_benchmarks.rs (run as tests)

These are benchmark-style tests in `crates/nine65/src/comprehensive_benchmarks.rs`. They run under `cargo test`, not `cargo bench`, so they produce human-readable table output rather than Criterion statistics.

### Run

```bash
cargo test -p nine65 --lib --release -- comprehensive_benchmarks --nocapture
```

### Tests

- **benchmark_noise_growth_secure_128**: Encrypts a value and performs 20 sequential `mul_public` operations, printing noise distance, collapse count, and per-operation time at each step.
- **benchmark_batch_operations_secure_128**: Encrypts 50 messages and measures batch timing.

---

## How to read Criterion output

When a Criterion benchmark completes, it prints a line like:

```
k_elimination_ct/extract_k/small
                        time:   [12.34 ns 12.56 ns 12.78 ns]
```

The three values inside the brackets are:

| Position | Meaning |
|----------|---------|
| Left (`12.34 ns`) | Lower bound of the 95% confidence interval |
| Center (`12.56 ns`) | Best estimate (point estimate) of the mean |
| Right (`12.78 ns`) | Upper bound of the 95% confidence interval |

If you see a line like `change: [-2.1234% -0.5678% +1.0123%]`, that is the percentage change from the previous run, also as a 95% confidence interval.

Criterion stores history in `target/criterion/`. To generate an HTML report:

```bash
cargo bench -p nine65 -- --output-format=html
```

Then open `target/criterion/report/index.html`.

---

## Performance baselines

These are the reference numbers from the current codebase on a standard CPU (no GPU):

| Config | Encrypt | Add | Mul | Decrypt |
|--------|---------|-----|-----|---------|
| secure_128 | 23.56 ms | 0.83 ms | 152.13 ms | 11.06 ms |
| secure_192 | 61.59 ms | 2.10 ms | 459.02 ms | 29.00 ms |

Depth chain performance:

| Config | Depth 50 total time |
|--------|---------------------|
| secure_128 | 6.29 s |
| secure_192 | 10.10 s |

RNS 4-lane raw arithmetic:

| Operation | Latency | Throughput |
|-----------|---------|------------|
| ADD | 65.7 ns | 15.2 M/s |
| MUL | 95.6 ns | 10.5 M/s |

---

## Where to go next

- [Testing](testing) -- how to run the full test suite and filter tests.
- [Extreme Tests](extreme-tests) -- adversarial boundary tests.
- [Security Configs](security-configs) -- parameters behind each config level.
