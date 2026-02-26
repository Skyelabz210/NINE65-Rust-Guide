---
layout: default
title: Entropy
parent: Using the System
nav_order: 7
---

# Entropy

[Home](index)

NINE65 provides three randomness sources: `ShadowHarvester` for fast deterministic entropy, `SecureRng` for OS CSPRNG, and `DeterministicRng` for ChaCha-based reproducible tests. All three implement the `FheRng` trait.

Source files: `entropy/shadow.rs`, `entropy/secure.rs`, `entropy/rng_trait.rs`, `entropy/crt_shadow.rs`, `entropy/shadow_entropy_monitor.rs`, `entropy/deterministic.rs`.

---

## FheRng Trait

The polymorphic RNG interface that all FHE operations accept.

```rust
pub trait FheRng {
    fn next_u64(&mut self) -> u64;
    fn uniform(&mut self, bound: u64) -> u64;
    fn ternary(&mut self) -> i64;           // {-1, 0, 1}
    fn cbd(&mut self, eta: usize) -> i64;   // [-eta, eta]
    fn ternary_vector(&mut self, n: usize) -> Vec<i64>;
    fn cbd_vector(&mut self, n: usize, eta: usize) -> Vec<i64>;
}
```

Both `ShadowHarvester` and `SecureRng` implement `FheRng`, so encryption and key generation functions can be generic over the randomness source:

```rust
// Works with any FheRng
ctx.encrypt_dual_with_rng(42, &pk, &mut rng);
ctx.generate_keys_dual_full_with_rng(&mut rng);
```

---

## ShadowHarvester

Defined in `entropy/shadow.rs`. Fast deterministic entropy via LFSR + counter mixing with MurmurHash3-style finalization. Achieves 5-10x faster than CSPRNGs while passing NIST SP 800-22 statistical tests.

### Construction

```rust
use nine65::entropy::ShadowHarvester;

// Default seed
let mut rng = ShadowHarvester::new();

// Specific seed for reproducibility
let mut rng = ShadowHarvester::with_seed(42);

// OS-seeded (fast sampling with non-deterministic seed)
let mut rng = ShadowHarvester::from_os_seed();

// Fallible OS-seeded
let mut rng = ShadowHarvester::try_from_os_seed()?;
```

### Core Operations

```rust
let val = rng.next_u64();           // 64 bits of entropy
let bits = rng.extract_bits(16);    // Extract n bits (1-64)
let bounded = rng.uniform(1000);    // Uniform in [0, 1000)
```

### Internal Algorithm

1. **LFSR step**: polynomial `x^64 + x^63 + x^61 + x^60 + 1`
2. **Counter increment**: monotonic 64-bit counter
3. **MurmurHash3 mixing**: XOR state with counter, multiply by golden ratio constant, triple XOR-shift finalization

### Thread Safety

`ShadowHarvester` is `Send` but **NOT** `Sync`. It maintains mutable internal state (LFSR and counter).

```rust
// WRONG: Do not share across threads
// let shared = Arc::new(Mutex::new(ShadowHarvester::new())); // Slow!

// RIGHT: Each thread gets its own harvester
let handles: Vec<_> = (0..4).map(|i| {
    std::thread::spawn(move || {
        let mut rng = ShadowHarvester::with_seed(42 + i);
        // Use rng locally
    })
}).collect();
```

### When to Use

- Testing and benchmarks (deterministic, reproducible)
- Evaluation key noise (where speed matters and CSPRNG is overkill)
- Encryption randomness in test environments

**Never use for production secret key generation.**

---

## SecureRng

Defined in `entropy/secure.rs`. Wraps the OS CSPRNG via the `getrandom` crate. Backed by `/dev/urandom` on Linux, `CryptGenRandom` on Windows.

### Construction

```rust
use nine65::entropy::SecureRng;

let mut rng = SecureRng::new();
```

### Operations

```rust
// Random bytes
let bytes = rng.random_bytes(32);
rng.fill_bytes(&mut buf);

// Random integers
let val = rng.random_u64();
let val = rng.random_u128();

// Bounded
let bounded = rng.random_u64_bounded(998244353);

// Fallible variants
let bytes = rng.try_random_bytes(32)?;
let val = rng.try_random_u64()?;
```

### When to Use

- **Secret key generation** (mandatory)
- **Public key randomness** (the `a` polynomial)
- **Any security-critical random values**
- **Shared contexts** (thread-safe, no mutable state concerns)

`SecureRng` is NIST SP 800-90B compliant.

---

## DeterministicRng

Defined in `entropy/deterministic.rs`. ChaCha20-based RNG for tests and benchmarks. Provides cryptographically strong output from a deterministic seed.

Available with the `deterministic_rng` feature flag or in test builds.

```rust
use nine65::entropy::DeterministicRng;

// From u64 seed (expanded via SplitMix64 to 32 bytes)
let mut rng = DeterministicRng::with_seed(42);

// From explicit 32-byte seed
let mut rng = DeterministicRng::from_seed([0u8; 32]);

let val = rng.next_u64();
let bits = rng.extract_bits(16);
let bounded = rng.uniform(1000);
```

**Not suitable for production secrecy** -- output is fully determined by the seed.

---

## CRT Shadow Entropy

Defined in `entropy/crt_shadow.rs`. Zero-cost entropy harvesting from computational byproducts.

**Coq-verified**: `CRTShadowEntropy.v`. **Lean-verified**: `ShadowEntropy.lean`.

### Core Concept

Every modular operation discards information (quotients):

```
a * b mod m  =>  quotient q = (a*b) / m   [discarded]
                 remainder r = (a*b) % m   [kept]
```

By Landauer's Principle, discarded information represents entropy. CRT Shadow captures these "shadows" and mixes them into cryptographic randomness.

### Entropy Yield

- Per modular reduction: ~12 bits (for 64-bit primes with uniform inputs)
- Per k-lane RNS multiply: ~12k bits raw, ~7-12 bits/byte after mixing
- At 2.4M ops/sec: ~86 Mbits/sec raw, ~8.6 Mbits/sec extracted

### Usage

```rust
use nine65::entropy::crt_shadow::{CRTShadowContext, ShadowAccumulator};

// Create context with RNS primes
let ctx = CRTShadowContext::new(&[998244353, 985661441]);
let mut acc = ShadowAccumulator::new();

// RNS multiply captures shadows
let a = ctx.from_int(12345);
let b = ctx.from_int(67890);
let (product, shadows) = ctx.mul_with_shadows(&a, &b);

// Ingest shadows for entropy
for s in shadows {
    acc.ingest(s);
}

// Extract random value
let random = acc.extract();
```

### ShadowAccumulator

Uses SipHash-inspired mixing with a 256-bit internal state (4 x u64 buffer). Tracks total bits ingested.

```rust
let mut acc = ShadowAccumulator::new();
acc.ingest(shadow_value);           // Feed in a shadow
let random = acc.extract();          // Extract mixed entropy
let bits = acc.bits_ingested();     // How many bits have been fed
```

---

## ShadowEntropyMonitor

Defined in `entropy/shadow_entropy_monitor.rs`. Monitors computational entropy byproducts and adapts system resources (thread count) accordingly.

Requires the `adaptive-threading` feature flag (which also requires `shadow-entropy`).

```rust
use nine65::entropy::ShadowEntropyMonitor;

let monitor = ShadowEntropyMonitor::new();
// Default: 4 threads, entropy threshold 1000, measurement interval 5
```

Fields:
- `entropy_level`: current measured entropy (AtomicU64)
- `entropy_threshold`: trigger threshold for adaptation
- `current_threads` / `max_threads` / `min_threads`: thread pool bounds
- `measurement_interval`: measure every Nth operation to reduce overhead
- `workload_counter` / `workload_threshold`: trigger adaptation after N operations
- `performance_history`: tracks (thread_count, duration) for optimization

The monitor is constant-time and float-free, suitable for secure FHE operations.

---

## Decision Tree: Which RNG to Use

```
Is this production key generation?
  YES -> SecureRng (mandatory)
  NO  -> Is this a test or benchmark?
           YES -> Need reproducibility?
                    YES -> ShadowHarvester::with_seed(FIXED)
                           or DeterministicRng::with_seed(FIXED)
                    NO  -> ShadowHarvester::from_os_seed()
           NO  -> Is this encryption randomness?
                    Production -> SecureRng or encrypt_*_secure()
                    Testing    -> ShadowHarvester
```

### Summary Table

| Operation | Recommended RNG |
|-----------|-----------------|
| Secret key generation | `SecureRng` (MANDATORY) |
| Public key generation | `SecureRng` (production) |
| Encryption randomness | `SecureRng` (production) or `ShadowHarvester` (testing) |
| Evaluation key noise | `ShadowHarvester` (speed) or `SecureRng` (paranoid) |
| Testing / benchmarks | `ShadowHarvester::with_seed()` or `DeterministicRng` |
| KAT vectors | `ShadowHarvester::with_seed(FIXED)` |

---

## Thread Safety Summary

| Type | Send | Sync | Notes |
|------|------|------|-------|
| `ShadowHarvester` | Yes | **No** | One per thread; do not share |
| `SecureRng` | Yes | Yes | Stateless wrapper; safe everywhere |
| `DeterministicRng` | Yes | **No** | One per thread |
| `ShadowAccumulator` | Yes | **No** | Per-computation-stream |
| `ShadowEntropyMonitor` | Yes | Yes | Uses atomics and mutexes internally |

---

## Where to go next

- [Key Management](key-management) -- how RNG choice affects key generation
- [MANA Accelerator](mana) -- entropy from parallel stream byproducts
- [Security Configs](security-configs) -- parameter choices that interact with entropy
