---
layout: default
title: Getting Started
parent: Foundation
nav_order: 1
---

# Getting Started

[Home](index)

Your first FHE computation in NINE65, from zero to encrypted arithmetic.

---

## Prerequisites

You need one thing: a Rust toolchain.

```bash
rustc --version    # 1.93.0 or later
cargo --version    # should match
```

If you do not have Rust installed, get it from [rustup.rs](https://rustup.rs). No Python, no Docker, no npm, no external C libraries. The entire system is pure Rust.

---

## Build the workspace

Clone the repo and build in release mode:

```bash
cd NINE65_v7
cargo build --release --workspace --exclude nine65-python --exclude nine65-wasm
```

The `--exclude` flags skip the Python and WASM binding crates, which need additional toolchains you do not need for core development. First build takes 1--2 minutes; subsequent builds take seconds.

Expected result: 0 errors. You may see a handful of warnings in `symmetric_bootstrap.rs` --- these are harmless unused-variable warnings and do not affect correctness.

---

## Run the system demo

```bash
cargo run -p nine65 --release --bin nine65_v7_demo
```

This binary is not a simulation. It imports directly from `nine65::` and exercises the real FHE pipeline end to end:

1. BFV single-modulus encrypt / decrypt / eval
2. K-Elimination DualRNS ciphertext-times-ciphertext multiplication
3. Clockwork Bootstrap across all three paths (circular, non-circular KSK, auto-triggered)
4. Kiosk architecture (Bullet / Capsule / Fuse self-destructing computation units)
5. Shadow Entropy deterministic reproducibility
6. Noise budget tracking
7. GSO-FHE depth tracking

Default seed is 42 (deterministic). For OS entropy:

```bash
cargo run -p nine65 --release --bin nine65_v7_demo -- --os-seed
```

For a custom seed:

```bash
cargo run -p nine65 --release --bin nine65_v7_demo -- --seed 12345
```

---

## Write your own: a first FHE computation

Create a file at `examples/my_first_fhe.rs` (or add a `#[test]` to an existing test file). Here is a complete working example that encrypts two values, adds them homomorphically, and decrypts the result:

```rust
use nine65::prelude::*;
use nine65::params::secure_configs::SecureConfig;
use nine65::entropy::ShadowHarvester;

fn main() {
    // 1. Choose a production-safe parameter set
    //    secure_128: n=4096, 3 primes, log2(q)=90, t=65537
    let secure = SecureConfig::secure_128();
    let config = secure.into_config();

    // 2. Build the RNS-native FHE context
    let ctx = RNSFHEContext::new(&config);

    // 3. Create a deterministic RNG (use SecureRng for production)
    let mut rng = ShadowHarvester::with_seed(42);

    // 4. Generate keys
    let keys = ctx.generate_keys_dual(&mut rng);

    // 5. Encrypt two plaintexts (must be < t = 65537)
    let a = 17u64;
    let b = 25u64;
    let ct_a = ctx.encrypt_dual(a, &keys.public_key, &mut rng);
    let ct_b = ctx.encrypt_dual(b, &keys.public_key, &mut rng);

    // 6. Homomorphic addition (operates on encrypted data)
    let ct_sum = ctx.add_dual(&ct_a, &ct_b);

    // 7. Decrypt
    let result = ctx.decrypt_dual(&ct_sum, &keys.secret_key);

    assert_eq!(result, a + b);
    println!("Encrypted {} + {} = {} (decrypted: {})", a, b, a + b, result);
}
```

Key points:

- `SecureConfig::secure_128()` gives you a verified 128-bit-secure parameter set. Never hand-roll parameters.
- `RNSFHEContext` is the recommended context for all FHE operations. It uses K-Elimination DualRNS internally.
- `ShadowHarvester::with_seed(42)` produces deterministic randomness --- good for testing and reproducibility. For production key generation and encryption, use `SecureRng::new()` or the `_secure()` method variants (e.g., `generate_keys_dual_secure()`, `encrypt_dual_secure()`).
- Plaintexts must be in the range `[0, t)` where `t = 65537`.

---

## Do changes reflect in demos and benchmarks?

Yes. Every binary in the project (`nine65_v7_demo`, `nine65_bench`, `fhe_demo`, `security_estimator_baseline`) imports from `nine65::` crate code. When you modify the library and rebuild, every binary and benchmark picks up your changes. There is no separate "demo mode" --- you are always running the real system.

```bash
# Edit some code in crates/nine65/src/...
# Then rebuild and re-run:
cargo run -p nine65 --release --bin nine65_v7_demo
```

---

## Where to go next

- [Building & Running](building) --- full build commands, all four binaries, the `--exclude` flags explained
- [Security Configs](security-configs) --- the three production parameter sets and what each number means
- [Cargo Reference](cargo-reference) --- every cargo command relevant to NINE65 development
