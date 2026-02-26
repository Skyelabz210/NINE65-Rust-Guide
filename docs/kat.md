---
layout: default
title: Known Answer Tests
parent: Tools & Testing
nav_order: 6
---

# Known Answer Tests (KAT)

[Home](index)

Known Answer Tests provide deterministic test vectors that verify FHE correctness across code changes, platform migrations, and compiler upgrades. If a KAT fails, something fundamental has changed in the arithmetic pipeline.

**Source**: `crates/nine65/src/kat.rs`

---

## What KATs are

A KAT is a fixed input-output pair: given a specific seed, parameters, plaintext, and operation, the system must always produce the exact same result. KATs serve three purposes:

1. **Regression testing** -- ensure updates do not break existing functionality.
2. **Cross-platform verification** -- confirm identical results on different CPUs, operating systems, and compilers. This is critical for NINE65's determinism guarantee.
3. **Certification compliance** -- support auditing and formal certification requirements where reproducible test vectors are mandatory.

---

## KATVector struct

Each test vector is defined by:

```rust
pub struct KATVector {
    pub name: &'static str,    // Test identifier
    pub seed: u64,             // Deterministic RNG seed
    pub n: usize,              // Ring dimension
    pub q: u64,                // Ciphertext modulus
    pub t: u64,                // Plaintext modulus
    pub plaintext: u64,        // Input plaintext value
    pub expected_result: u64,  // Expected decrypted output
    pub operation: KATOperation,
}
```

The seed controls all randomness: key generation uses `ShadowHarvester::with_seed(seed)`, and encryption uses `ShadowHarvester::with_seed(seed + 1)`. This means the exact same ciphertext bytes are produced every run.

---

## KATOperation enum

Four operation types are supported:

| Variant | Description | Expected result |
|---------|-------------|-----------------|
| `EncryptDecrypt` | Encrypt plaintext, then decrypt. | `expected_result == plaintext` |
| `HomomorphicAdd { addend }` | Encrypt plaintext, add `addend` as plaintext, decrypt. | `expected_result == (plaintext + addend) mod t` |
| `MulPlain { multiplier }` | Encrypt plaintext, multiply by `multiplier` as plaintext, decrypt. | `expected_result == (plaintext * multiplier) mod t` |
| `CtCtAdd { other }` | Encrypt both `plaintext` and `other`, add ciphertexts, decrypt. | `expected_result == (plaintext + other) mod t` |

---

## STANDARD_KATS constant

The crate ships with 8 built-in test vectors in `STANDARD_KATS`:

| Name | Seed | Operation | Plaintext | Expected |
|------|------|-----------|-----------|----------|
| `encrypt_decrypt_zero` | `0xDEADBEEF` | EncryptDecrypt | 0 | 0 |
| `encrypt_decrypt_one` | `0xDEADBEEF` | EncryptDecrypt | 1 | 1 |
| `encrypt_decrypt_42` | `0xDEADBEEF` | EncryptDecrypt | 42 | 42 |
| `encrypt_decrypt_max` | `0xDEADBEEF` | EncryptDecrypt | 2052 (t-1) | 2052 |
| `add_plain_100_plus_50` | `0x12345678` | HomomorphicAdd(50) | 100 | 150 |
| `mul_plain_17_times_3` | `0x87654321` | MulPlain(3) | 17 | 51 |
| `ct_ct_add_25_plus_75` | `0xCAFEBABE` | CtCtAdd(75) | 25 | 100 |
| `encrypt_decrypt_different_seed` | `0x11111111` | EncryptDecrypt | 123 | 123 |

All vectors use N=1024, q=998244353, t=2053 (parameters chosen for fast execution while still exercising the full BFV pipeline).

---

## KATResult struct

Each KAT run produces:

```rust
pub struct KATResult {
    pub name: String,
    pub passed: bool,
    pub expected: u64,
    pub actual: u64,
    pub ct_hash: Option<[u8; 32]>,  // SHA-256 of ciphertext c0 coefficients
}
```

The `ct_hash` field is populated only for `EncryptDecrypt` operations. It provides a fingerprint of the ciphertext itself (not just the decrypted result), which catches changes in the encryption randomness path even if decryption still succeeds.

---

## API functions

### run_kat(kat: &KATVector) -> KATResult

Runs a single KAT vector. Internally:
1. Creates an `FHEConfig::custom(n, vec![q], t, 2)`.
2. Initializes `NTTEngine`, `ShadowHarvester`, and `KeySet::generate`.
3. Creates `BFVEncoder`, `BFVEncryptor`, `BFVDecryptor`.
4. Uses a separate `ShadowHarvester::with_seed(seed + 1)` for encryption.
5. Executes the operation and decrypts.
6. Compares actual result against expected.

### run_all_kats() -> Vec\<KATResult\>

Runs all `STANDARD_KATS` and collects results.

### all_kats_passed() -> bool

Convenience: returns true only if every standard KAT passes.

### print_kat_results(results: &[KATResult])

Prints a formatted table with pass/fail status for each vector.

---

## When to run KATs

Run KATs in these situations:

| Situation | Why |
|-----------|-----|
| After modifying arithmetic code | K-Elimination, NTT, Barrett, Montgomery, or BFV encode/decode changes could silently alter results. |
| After modifying the entropy system | `ShadowHarvester` changes affect key generation and encryption randomness. |
| Before tagging a release | KATs are the final gate confirming the release matches known-good behavior. |
| On a new platform or compiler version | Confirms bit-identical behavior across environments. |
| After dependency updates | Ensures no upstream change affects the arithmetic pipeline. |

---

## Running KATs

### As part of the test suite

```bash
cargo test -p nine65 --lib --release -- kat::tests --nocapture
```

This runs four test functions:

- `test_all_kats` -- runs all 8 vectors, prints the results table, and asserts all pass.
- `test_kat_deterministic` -- runs the same KAT vector twice and asserts identical `actual` values and `ct_hash` values.
- `test_kat_encrypt_decrypt` -- runs only EncryptDecrypt vectors.
- `test_kat_homomorphic_ops` -- runs only homomorphic operation vectors.

### Quick check

```bash
cargo test -p nine65 --lib --release -- test_all_kats --nocapture
```

### Example output

```
+---------------------------------------------------------------+
|                    QMNF FHE Known Answer Tests                |
+---------------------------------------------------------------+
| PASS   | encrypt_decrypt_zero                                 |
| PASS   | encrypt_decrypt_one                                  |
| PASS   | encrypt_decrypt_42                                   |
| PASS   | encrypt_decrypt_max                                  |
| PASS   | add_plain_100_plus_50                                |
| PASS   | mul_plain_17_times_3                                 |
| PASS   | ct_ct_add_25_plus_75                                 |
| PASS   | encrypt_decrypt_different_seed                       |
+---------------------------------------------------------------+
| Results: 8 passed, 0 failed                                  |
+---------------------------------------------------------------+
```

---

## Adding custom KAT vectors

To add a new KAT, define a `KATVector` and call `run_kat`:

```rust
use nine65::kat::{KATVector, KATOperation, run_kat};

let custom = KATVector {
    name: "my_custom_test",
    seed: 0xABCD1234,
    n: 1024,
    q: 998244353,
    t: 2053,
    plaintext: 500,
    expected_result: 1500,
    operation: KATOperation::MulPlain { multiplier: 3 },
};

let result = run_kat(&custom);
assert!(result.passed, "Custom KAT failed: expected {}, got {}", result.expected, result.actual);
```

When adding vectors, choose seeds that differ from existing ones to ensure independent randomness streams.

---

## Where to go next

- [Testing](testing) -- the full test suite and how KATs fit into the broader test strategy.
- [Benchmarks](benchmarks) -- performance measurement (KATs verify correctness, benchmarks verify speed).
- [Security Configs](security-configs) -- the production parameter sets (KATs use small test parameters for speed).
