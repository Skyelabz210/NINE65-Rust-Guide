---
layout: default
title: Feature Flags
parent: Foundation
nav_order: 6
---

# Feature Flags

[Home](index)

The `nine65` crate defines feature flags in `crates/nine65/Cargo.toml`. Each flag gates optional modules, dependencies, or behavioral changes. Default features are `ntt_fft` and `exact_transcendentals_backend`.

---

## Default Features

### ntt_fft

**Default: on.**

Enables the FFT-based NTT engine (`NTTEngineFFT`) and re-exports it as the primary `NTTEngine` in the prelude.

- When enabled: `use nine65::prelude::NTTEngine` resolves to `NTTEngineFFT`
- When disabled: `use nine65::prelude::NTTEngine` resolves to the DFT-based `NTTEngine`
- Both engines are always available by their explicit names (`NTTEngineFFT`, `NTTEngineDFT`)

The FFT-based engine is faster for large polynomial degrees (n >= 4096). There is no reason to disable this unless you are specifically benchmarking the DFT engine.

```bash
cargo test -p nine65 --lib --release                          # uses ntt_fft (default)
cargo test -p nine65 --lib --release --no-default-features    # uses DFT engine
```

### exact_transcendentals_backend

**Default: on.**

Pulls in the `exact_transcendentals` crate, which provides integer-only CORDIC and AGM implementations of trigonometric, exponential, and logarithmic functions. Without this flag, any code path that calls into exact transcendentals will fail to compile.

```bash
cargo build -p nine65 --release                               # includes exact_transcendentals
cargo build -p nine65 --release --no-default-features --features ntt_fft  # excludes it
```

---

## Acceleration Flags

### accelerated

Enables the MANA (Modular Anchored Number Arithmetic) stream accelerator and UNHAL (hardware abstraction layer) crates. This is the recommended parallelism path for NINE65.

- Adds dependencies: `mana`, `unhal`
- Makes available: `nine65::accelerated` module
- MANA is purpose-built for this system's RNS lane structure. It outperforms generic Rayon for this workload.

```bash
cargo build -p nine65 --release --features accelerated
cargo test -p nine65 --lib --release --features accelerated
```

### parallel

Enables Rayon-based opt-in parallelism. This is the generic alternative to MANA.

- Adds dependency: `rayon`
- Makes available: parallel encryptor/decryptor and Rayon-based NTT
- Required for the `throughput` and `adaptive_rayon` benchmarks

```bash
cargo bench -p nine65 --bench throughput --features parallel
cargo bench -p nine65 --bench adaptive_rayon --features parallel
```

For most use cases, prefer `--features accelerated` over `--features parallel`.

### sequential

Forces sequential execution. Disables all parallelism, including MANA. Useful for determinism testing and debugging race conditions.

```bash
cargo test -p nine65 --lib --release --features sequential
```

---

## Entropy Flags

### shadow-entropy

Enables the CRT Shadow entropy subsystem and WASSAN noise field. Shadow entropy is a proprietary entropy measurement technique based on Landauer's principle and Shannon information theory, applied to monitor computational noise state in FHE operations.

- Makes available: `nine65::entropy::crt_shadow`, `nine65::entropy::wassan_noise`
- Exports `WassanNoiseField` in the prelude (behind this flag)
- Required by `adaptive-threading`

```bash
cargo build -p nine65 --release --features shadow-entropy
```

### adaptive-threading

Entropy-driven adaptive thread management. Adjusts parallelism based on shadow-entropy state monitoring. Requires deep familiarity with the QMNF architecture.

- Requires: `shadow-entropy` (automatically enabled)
- Internal subsystem, not typically enabled for regular development

```bash
cargo build -p nine65 --release --features adaptive-threading
```

### deterministic_rng

Enables a ChaCha20-based deterministic CSPRNG for reproducible testing.

- Adds dependencies: `rand_chacha`, `rand_core`
- Makes available: `nine65::entropy::DeterministicRng`
- Exports `DeterministicRng` in the prelude (behind this flag)

Note: `DeterministicRng` is also available in `#[cfg(test)]` without this flag. The flag makes it available in non-test code.

```bash
cargo build -p nine65 --release --features deterministic_rng
```

---

## Arithmetic Flags

### exact_rational

Bridges the `nexgen_rational` crate into nine65 for exact i128 rational arithmetic. Used for precise BFV delta computation and noise budget calculations.

- Adds dependency: `nexgen_rational`
- Enables exact rational noise tracking paths

```bash
cargo test -p nine65 --lib --release --features exact_rational
```

### clockwork

Enables GRO (Galois Ring Oscillator) timing gates, formal bound tracking, key lifecycle integrity checks, and Garner reconstruction from the `clockwork-core` crate.

- Adds dependencies: `clockwork-core`, `crc32fast`
- Enables cross-validation of RNS operations against clockwork bounds
- Required for: `clockwork_cross_validation` integration test

```bash
cargo test -p nine65 --test clockwork_cross_validation --release --features clockwork
```

---

## Serialization

### serde

Enables JSON and bincode serialization for ciphertexts, keys, and configs.

- Adds dependencies: `serde`, `serde_json`, `bincode`
- Adds `#[derive(Serialize, Deserialize)]` to key types like `RNSSecretKey`, `DualRNSCiphertext`
- Required for the `nine65_bench` binary

```bash
cargo run -p nine65 --release --bin nine65_bench --features serde
cargo test -p nine65 --lib --release --features serde
```

---

## Testing Flags

### allow_insecure

Makes insecure test configurations (`test_fast_insecure`, `test_medium_insecure`) available in non-test, non-debug builds. Also relaxes the runtime security assertion in `new_verified()`.

- Required for: `nine65-extreme-tests`, `fhe_demo` with `--config light`
- Without this flag, release builds cannot construct configs with fewer than 128 bits of security

```bash
cargo test -p nine65 --lib --release --features allow_insecure
```

**Never ship production code with this flag.** It exists solely for testing and benchmarking with fast parameters.

### extreme-tests

Defined in the `nine65-extreme-tests` crate (not in `nine65` itself). Gates expensive boundary and adversarial tests.

```bash
cargo test -p nine65-extreme-tests --features extreme-tests --release
```

### slow_tests / benchmarks

Gate expensive test and benchmark code paths within the `nine65` crate. The `timing` bench requires `benchmarks`.

```bash
cargo bench -p nine65 --bench timing --features benchmarks
```

---

## Compound / Convenience Flags

### v2

Shorthand for `ntt_fft` + `wassan` (which itself enables `shadow-entropy`). Turns on the FFT NTT engine and the WASSAN noise field together.

```bash
cargo build -p nine65 --release --features v2
```

### wassan

Shorthand for `shadow-entropy`. Enables the WASSAN noise field subsystem.

---

## Common Combinations

| Use Case | Command |
|----------|---------|
| **Normal development** | `cargo build -p nine65 --release` (defaults: `ntt_fft`, `exact_transcendentals_backend`) |
| **Full test suite** | `cargo test -p nine65 --lib --release` |
| **With insecure test configs** | `cargo test -p nine65 --lib --release --features allow_insecure` |
| **Clockwork validation** | `cargo test -p nine65 --release --features clockwork` |
| **Exact rational noise** | `cargo test -p nine65 --lib --release --features exact_rational` |
| **Serialization tests** | `cargo test -p nine65 --lib --release --features serde` |
| **MANA acceleration** | `cargo build -p nine65 --release --features accelerated` |
| **Rayon benchmarks** | `cargo bench -p nine65 --bench throughput --features parallel` |
| **Extreme boundary tests** | `cargo test -p nine65-extreme-tests --features extreme-tests --release` |
| **Everything except insecure** | `cargo test -p nine65 --release --features "clockwork,exact_rational,serde,accelerated"` |
| **Kitchen sink (testing)** | `cargo test -p nine65 --release --features "allow_insecure,clockwork,exact_rational,serde,shadow-entropy,deterministic_rng"` |

---

## Dependency Graph Between Flags

```
accelerated ──> mana, unhal
shadow-entropy (standalone)
adaptive-threading ──> shadow-entropy
wassan ──> shadow-entropy
v2 ──> ntt_fft, wassan ──> shadow-entropy
clockwork ──> clockwork-core, crc32fast
exact_rational ──> nexgen_rational
exact_transcendentals_backend ──> exact_transcendentals
parallel ──> rayon
generic-rayon ──> rayon
deterministic_rng ──> rand_chacha, rand_core
serde ──> serde, serde_json, bincode
```

Flags that add no external dependencies: `ntt_fft`, `allow_insecure`, `slow_tests`, `benchmarks`, `sequential`, `secure-keygen`, `secure_seed`, `debug_dual_mul`.

---

## Where to go next

- [Cargo Reference](cargo-reference) --- how to pass feature flags to build, test, and bench commands
- [Rust Patterns](rust-patterns) --- how `#[cfg(feature = "...")]` gating works in the source
- [Security Configs](security-configs) --- how `allow_insecure` relates to production vs test configs
