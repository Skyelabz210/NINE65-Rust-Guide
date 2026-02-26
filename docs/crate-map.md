---
layout: default
title: Crate Map
parent: How It Works
nav_order: 14
---

# Crate Map


## Workspace Layout

```
crates/
├── nine65/                    # Core FHE library (721 tests)
├── clockwork-core/            # Formal-spec RNS (46 tests)
├── exact_transcendentals/     # Exact CORDIC transcendentals (143 tests)
├── nexgen_rational/           # Exact i128 rational arithmetic (95 tests)
├── fhe-service/               # HTTP session management (22 tests)
├── mana/                      # FHE stream accelerator (30 tests)
├── unhal/                     # Hardware abstraction layer (10 tests)
└── nine65-extreme-tests/      # Boundary/adversarial harness (85 tests)
```

## nine65 (Core)

The main library. Everything FHE lives here.

### Key directories

| Path | What's there |
|------|-------------|
| `src/arithmetic/` | RNS, K-Elimination, NTT, Montgomery, U256 |
| `src/ops/` | FHE operations: encrypt, decrypt, add, mul, bootstrap |
| `src/noise/` | Noise budget tracking (millibits precision) |
| `src/params/` | Configs, security estimator, validation |
| `src/keys/` | Key generation: secret, public, BSK, KSK, eval |
| `src/entropy/` | CRT shadow harvester, CSPRNG, adaptive threading |
| `src/security/` | Constant-time primitives, GRO gates |
| `src/bin/` | Four binaries (demo, bench, estimator, fhe_demo) |

### Key files

| File | What it does |
|------|-------------|
| `ops/rns_fhe.rs` | BFV encrypt/decrypt/add/mul --- the main FHE operations |
| `ops/bootstrap.rs` | Clockwork Bootstrap --- three-phase noise reset |
| `ops/auto_bootstrap.rs` | AutoBootstrapEvaluator --- automatic noise-triggered bootstrap |
| `ops/gso_fhe.rs` | GSO depth management and benchmarks |
| `arithmetic/rns.rs` | DualRNS context, CRT reconstruction, U256 |
| `arithmetic/k_elimination.rs` | K-Elimination: exact CRT with anchor primes |
| `arithmetic/ntt_fft.rs` | Number Theoretic Transform (Cooley-Tukey) |
| `arithmetic/montgomery.rs` | Montgomery modular arithmetic (constant-time) |
| `noise/budget.rs` | NoiseBudget: millibits tracking, consume/reset |
| `params/secure_configs.rs` | SecureConfig: validated production parameters |
| `params/security_estimator.rs` | LatticeSecurityEstimator: Core-SVP + MATZOV |
| `entropy/shadow.rs` | ShadowHarvester: deterministic seeded RNG |

## clockwork-core

Formal-specification RNS library. Provides Garner reconstruction, GRO timing gates, bound tracking, and integrity verification. Used by nine65 for formal correctness guarantees.

## exact_transcendentals

Exact transcendental functions (sin, cos, exp, log) via integer CORDIC algorithm. No floating-point. Used when FHE applications need trigonometric or exponential operations.

## nexgen_rational

Exact rational arithmetic using i128 numerator/denominator. Zero dependencies. Used for exact noise calculations and BFV delta computation when the `exact_rational` feature is enabled.

## fhe-service

HTTP session management layer. Serializes/deserializes FHE contexts and ciphertexts for network transport. Used by the Cloud Run deployment.

## mana

FHE stream accelerator. Lane-parallel pipeline engine that processes batches of FHE operations. The canonical parallelism strategy (Rayon is opt-in, MANA is the default architecture).

## unhal

Hardware Abstraction Layer. Abstracts CPU feature detection and platform-specific optimizations. Thin layer between nine65 and the hardware.

## nine65-extreme-tests

Adversarial and boundary testing harness. 85 tests across 14 modules. Opt-in via `--features extreme-tests`. Answers questions the normal test suite does not ask. See [Extreme Tests](extreme-tests).
