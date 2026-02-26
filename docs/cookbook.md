---
layout: default
title: Cookbook
parent: Using the System
nav_order: 1
---

# Cookbook

[Home](index)

Code-first recipes for common NINE65 FHE operations. Each recipe shows the minimal working code path.

---

## 1. Basic BFV Roundtrip

Encode a scalar, encrypt it, perform a homomorphic add, then decrypt.

```rust
use nine65::params::FHEConfig;
use nine65::keys::{SecretKey, PublicKey, KeySet};
use nine65::ops::encrypt::{BFVEncoder, BFVEncryptor, BFVDecryptor};
use nine65::entropy::ShadowHarvester;

// Setup
let config = FHEConfig::default_test(); // or a secure config
let ntt = nine65::arithmetic::NTTEngineFFT::new(config.n, config.q);
let mut rng = ShadowHarvester::with_seed(42);

// Key generation
let keys = KeySet::generate(&config, &ntt, &mut rng);

// Encode and encrypt
let encoder = BFVEncoder::new(&config);
let encryptor = BFVEncryptor::new(&config, &ntt);
let decryptor = BFVDecryptor::new(&config, &ntt);

let m = 7u64;
let pt = encoder.encode(m);
let ct = encryptor.encrypt(&pt, &keys.public_key, &mut rng);

// Homomorphic add with another ciphertext
let m2 = 3u64;
let pt2 = encoder.encode(m2);
let ct2 = encryptor.encrypt(&pt2, &keys.public_key, &mut rng);
let ct_sum = ct.add(&ct2);

// Decrypt
let result_poly = decryptor.decrypt(&ct_sum, &keys.secret_key);
let result = encoder.decode(&result_poly);
assert_eq!(result, (m + m2) % config.t);
```

---

## 2. DualRNS FHE (Production Path)

The recommended path for all real workloads. Uses `SecureConfig`, DualRNS K-Elimination, and public-mode multiplication.

```rust
use nine65::params::secure_configs::SecureConfig;
use nine65::ops::rns_fhe::RNSFHEContext;
use nine65::entropy::ShadowHarvester;

// 1. Create context from secure parameters
let sc = SecureConfig::secure_128();
let config = sc.into_config();
let ctx = RNSFHEContext::new(&config);

// 2. Generate full keys (sk + pk + eval key)
let mut rng = ShadowHarvester::with_seed(42);
let keys = ctx.generate_keys_dual_full(&mut rng);
// Production: let keys = ctx.generate_keys_dual_full_secure();

// 3. Encrypt two values
let ct_a = ctx.encrypt_dual(7, &keys.public_key, &mut rng);
let ct_b = ctx.encrypt_dual(3, &keys.public_key, &mut rng);

// 4. Homomorphic multiply (public mode -- no secret key needed)
let ct_prod = ctx.mul_dual_public(&ct_a, &ct_b, &keys.eval_key)
    .expect("multiplication should succeed");

// 5. Decrypt
let result = ctx.decrypt_dual(&ct_prod, &keys.secret_key);
assert_eq!(result, (7 * 3) % config.t);
```

**For production**, replace `ShadowHarvester` calls with `_secure()` variants:

```rust
let keys = ctx.generate_keys_dual_full_secure();
let ct = ctx.encrypt_dual_secure(42, &keys.public_key);
```

---

## 3. Auto-Bootstrap (Unlimited Depth)

Chain 10+ multiplications without manual bootstrap management. The evaluator triggers bootstrapping automatically when the noise budget drops below 25%.

```rust
use nine65::params::secure_configs::SecureConfig;
use nine65::ops::rns_fhe::RNSFHEContext;
use nine65::ops::bootstrap::ClockworkBootstrap;
use nine65::ops::auto_bootstrap::AutoBootstrapEvaluator;
use nine65::keys::bootstrap::BootstrapKey;
use nine65::entropy::ShadowHarvester;

let sc = SecureConfig::secure_128();
let config = sc.into_config();
let ctx = RNSFHEContext::new(&config);

let mut rng = ShadowHarvester::with_seed(42);
let keys = ctx.generate_keys_dual_full(&mut rng);

// Setup bootstrap infrastructure
let bootstrap = ClockworkBootstrap::new(&config).expect("bootstrap setup");
let bsk_set = bootstrap.generate_keys(&config, &ctx, &keys, &mut rng)
    .expect("bootstrap keygen");

// Create auto-bootstrap evaluator
let mut evaluator = AutoBootstrapEvaluator::new(
    &ctx,
    &bootstrap,
    &bsk_set.bsk,
    &bsk_set.ksk,
    &keys.eval_key,
    &config,
);

// Optionally tune the trigger threshold (default: 250 = 25%)
evaluator.set_trigger_threshold(250);

// Encrypt a value
let ct = ctx.encrypt_dual(2, &keys.public_key, &mut rng);

// Chain 10+ multiplications -- bootstrap fires automatically
let mut result = ct.clone();
for _ in 0..10 {
    result = evaluator.mul_auto(&result, &ct).expect("auto mul");
}

// Check stats
println!("{}", evaluator.budget_summary());
// e.g. "Noise Budget: 15/30 bits remaining | bootstraps: 3, muls: 10, adds: 0"

let plaintext = ctx.decrypt_dual(&result, &keys.secret_key);
// plaintext == 2^10 mod t
```

---

## 4. Batch Encoding

Pack up to N values into a single ciphertext. Homomorphic addition works coefficient-wise (SIMD-like). Multiplication produces a polynomial product (convolution), not element-wise product.

```rust
use nine65::ops::batch::BatchEncoder;
use nine65::params::FHEConfig;

let config = FHEConfig { n: 1024, t: 65537, q: 132120577, ..Default::default() };
let encoder = BatchEncoder::new(&config).expect("encoder");

// Encode up to N=1024 values
let values_a = vec![10u64, 20, 30, 40];
let values_b = vec![1u64, 2, 3, 4];

let poly_a = encoder.encode(&values_a).expect("encode a");
let poly_b = encoder.encode(&values_b).expect("encode b");

// After homomorphic addition of the encrypted polynomials:
//   result coefficients = (10+1, 20+2, 30+3, 40+4, 0+0, ...)
let decoded = encoder.decode(&result_poly, values_a.len());
// decoded == [11, 22, 33, 44]

// Capacity check
assert_eq!(encoder.capacity(), 1024);

// SIMD slot batching requires t ≡ 1 (mod 2N) -- future enhancement
let has_simd = BatchEncoder::supports_simd_slots(&config);
```

---

## 5. Noise Budget Inspection

Track how much computational headroom remains before decryption would fail.

```rust
use nine65::noise::budget::{NoiseBudget, NoiseOpType};
use nine65::params::secure_configs::SecureConfig;

let config = SecureConfig::secure_128().into_config();

// Create budget from config
let mut budget = NoiseBudget::from_config(&config);
println!("Initial:  {} millibits", budget.remaining_millibits());
println!("Can do ~{} muls", budget.remaining_multiplications(&config));

// Simulate operations
let mul_cost = NoiseBudget::mul_ct_cost(&config)
             + NoiseBudget::relin_cost(&config);
budget.consume(NoiseOpType::MulCt, mul_cost).expect("budget ok");

println!("After mul: {} millibits", budget.remaining_millibits());

// Check if bootstrap is needed (threshold = 250 permille = 25%)
if budget.should_bootstrap(250) {
    println!("Time to bootstrap!");
}

// Full summary
println!("{}", budget.summary());
```

Cost estimates available:

| Method | What it measures |
|--------|-----------------|
| `NoiseBudget::encrypt_cost(config)` | Fresh encryption noise |
| `NoiseBudget::add_cost()` | ct + ct (1000 millibits) |
| `NoiseBudget::mul_ct_cost(config)` | ct * ct |
| `NoiseBudget::relin_cost(config)` | Relinearization after mul |
| `NoiseBudget::rescale_cost(config)` | K-Elimination rescale (negative -- gives back budget) |
| `NoiseBudget::multiplication_cycle_cost(config)` | mul + relin + rescale combined |

---

## 6. GSO-FHE (Basin-Based Depth Management)

Gravitational Swarm Optimization wraps `RNSFHEContext` with noise bounding via attractor basins. When noise exceeds the basin radius, a collapse operation reconverges.

```rust
use nine65::ops::gso_fhe::{GSOFHEContext, GSOCiphertext};
use nine65::ops::rns_fhe::RNSFHEContext;
use nine65::params::secure_configs::SecureConfig;
use nine65::entropy::ShadowHarvester;

let config = SecureConfig::secure_128().into_config();
let inner = RNSFHEContext::new(&config);
let gso = GSOFHEContext::new(inner);

let mut rng = ShadowHarvester::with_seed(42);
let keys = gso.generate_full_keys(&mut rng);

// Encrypt with GSO tracking
let ct = gso.encrypt(7, &keys.public_key, &mut rng);
println!("depth={}, noise_distance={}", ct.depth(), ct.noise_distance());

// Multiply -- noise and depth are tracked automatically
let ct2 = gso.encrypt(3, &keys.public_key, &mut rng);
let ct_prod = gso.mul(&ct, &ct2, &keys.eval_key, &mut rng)
    .expect("gso mul");
println!("depth={}, collapses={}", ct_prod.depth(), ct_prod.collapses());

// Decrypt through the inner context
let result = gso.inner.decrypt_dual(&ct_prod.inner, &keys.secret_key);
```

---

## 7. Key Generation: `generate()` vs `generate_secure()`

Every key type has two constructors. Pick the right one.

```rust
use nine65::keys::{SecretKey, PublicKey, KeySet};
use nine65::entropy::ShadowHarvester;

// ==========================================
// TESTING: deterministic, reproducible
// ==========================================
let mut rng = ShadowHarvester::with_seed(42);
let sk = SecretKey::generate(&config, &mut rng);
let pk = PublicKey::generate(&sk, &config, &ntt, &mut rng);

// ==========================================
// PRODUCTION: OS CSPRNG, non-deterministic
// ==========================================
let sk = SecretKey::generate_secure(&config);
let pk = PublicKey::generate_secure(&sk, &config, &ntt);

// DualRNS variants follow the same pattern:
let ctx = RNSFHEContext::new(&config);
// Testing:
let keys = ctx.generate_keys_dual_full(&mut rng);
// Production:
let keys = ctx.generate_keys_dual_full_secure();
```

**Rule**: Never use `generate()` / `ShadowHarvester` for real key material. Shadow entropy is deterministic and NOT cryptographically secure for key generation.

---

## 8. Known Answer Tests (KAT)

Run deterministic test vectors to verify correctness across builds and platforms.

```rust
use nine65::kat::{run_all_kats, KATResult};

let results = run_all_kats();
for r in &results {
    println!("{}: {}", r.name, if r.passed { "PASS" } else { "FAIL" });
    if !r.passed {
        println!("  expected={}, actual={}", r.expected, r.actual);
    }
}

// All should pass
assert!(results.iter().all(|r| r.passed));
```

KAT vectors cover: encrypt/decrypt roundtrip, homomorphic addition, plaintext multiplication, and ciphertext-ciphertext addition. Each uses a fixed seed for deterministic reproducibility.

---

## Where to go next

- [Key Management](key-management) -- key types, bootstrap keys, memory safety
- [Noise Budget](noise-budget) -- deep dive on millibits tracking
- [Bootstrap System](bootstrap) -- how Clockwork Bootstrap works under the hood
