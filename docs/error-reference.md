---
layout: default
title: Error Reference
parent: Reference
nav_order: 1
---

# Error Reference

[Home](index)

Every error in NINE65 maps to a specific formal precondition violation. This page documents every `Nine65Error` variant: what it means, what causes it, which theorem it traces to, and what to do about it.

**Source**: `crates/nine65/src/errors.rs`

---

## Error type overview

All errors are variants of `Nine65Error`, which derives `Debug`, `Clone`, and `thiserror::Error`. The companion type alias `Nine65Result<T>` is `Result<T, Nine65Error>`.

Two helper methods on `Nine65Error`:

- `is_recoverable()` -- returns true for `NoiseOverflow`, `DepthExceeded`, `BootstrapFailed`, and `NoiseBudgetExhausted`. These indicate the operation can be retried after a bootstrap refresh.
- `is_batching_error()` -- returns true for `BatchingNotSupported`, `TooManySlotValues`, and `NoModularInverse`.
- `category()` -- returns a static string label for logging: "K-Elimination", "GSO-FHE", "Order Finding", "Arithmetic", "Cryptographic", "Encoding", "Configuration", "Batching", "Serialization", "Bootstrap", "Entropy", "Noise", or "Regime".

---

## K-Elimination errors

These map to preconditions in `KElimination.v` (Lean4 canonical) and `ExactCoefficient.v`.

### NotCoprime

```
coprimality violation: gcd({m}, {a}) = {gcd} != 1
```

**Cause**: The main modulus M and anchor modulus A share a common factor. K-Elimination requires gcd(M, A) = 1 to compute the modular inverse `M^(-1) mod A`.

**Theorem**: `KElimination.v:kElimination_core` -- coprime moduli precondition.

**Fix**: Use coprime moduli. In practice, this means your RNS basis contains moduli that are not pairwise coprime. Check that all primes in your basis are distinct primes. If using `DualRNSContext::for_fhe()`, this should never happen with standard configurations.

### RangeOverflow

```
range overflow: X={x} >= M*A={bound}
```

**Cause**: The value X being reconstructed exceeds the product M*A. K-Elimination can only recover values in [0, M*A).

**Theorem**: `KElimination.v:kElimination_core` -- `X < M * A` precondition.

**Fix**: The intermediate values in your computation have grown too large for the current anchor capacity. This was the root cause of the anchor capacity fix (commit `73a372e`): upgrading from 3 anchors (~94-bit product) to 5 anchors (~158-bit product) resolved this for ct*ct multiplication. If you hit this on deep circuits, use bootstrap to refresh before the intermediate exceeds capacity.

### ModulusZero

```
modulus zero: M must be > 0
```

**Cause**: A zero was passed as a modulus value.

**Theorem**: `KElimination.v` -- `M > 0` precondition.

**Fix**: Check your configuration. This usually indicates an uninitialized or default-constructed config.

### AnchorZero

```
anchor zero: A must be > 0
```

**Cause**: A zero was passed as an anchor modulus value.

**Theorem**: `KElimination.v` -- `A > 0` precondition.

**Fix**: Same as ModulusZero -- check config construction.

### InexactDivision

```
inexact division: {value} not divisible by {divisor}
```

**Cause**: An exact division was attempted but the value is not evenly divisible by the divisor.

**Theorem**: `ExactCoefficient.v:div_exact` -- divisibility precondition.

**Fix**: This indicates a logic error in the BFV rescaling path. The delta-scaling step should always produce values divisible by the scale factor. If you see this, there may be a noise corruption issue upstream.

---

## GSO-FHE errors

These map to bounds in `GSOFHE.v`.

### NoiseOverflow

```
noise overflow: {level} > threshold {threshold}
```

**Cause**: The noise level in a ciphertext has exceeded the collapse threshold. Decryption would produce garbage.

**Theorem**: `GSOFHE.v:noise_bounded` -- noise must stay below threshold.

**Recoverable**: Yes. Bootstrap the ciphertext to refresh its noise budget.

**Fix**: Either reduce circuit depth, insert a bootstrap call before the overflow point, or use `AutoBootstrapEvaluator` which triggers bootstrap automatically when noise approaches the threshold.

### DepthExceeded

```
depth exceeded: {depth} > max {max_depth}
```

**Cause**: The circuit has exceeded the maximum supported depth for the current configuration.

**Theorem**: `GSOFHE.v:depth_50_achievable` -- depth bounds.

**Recoverable**: Yes. Bootstrap resets the depth counter.

**Fix**: Use a deeper configuration (`secure_128_deep`), use bootstrap, or restructure the circuit to reduce multiplicative depth.

---

## Order Finding errors

These map to `OrderFinding.v`.

### OrderNotFound

```
order not found: a={a} mod N={n} within bound {bound}
```

**Cause**: The multiplicative order of `a` modulo `N` was not found within the search bound.

**Theorem**: `OrderFinding.v:lagrange_bound` -- order must be <= N-1.

**Fix**: Increase the search bound, or verify that `a` and `N` are coprime (see `NotCoprimeToModulus`).

### NotCoprimeToModulus

```
not coprime to modulus: gcd({a}, {n}) != 1
```

**Cause**: The base `a` shares a factor with the modulus `N`. Multiplicative order is only defined when gcd(a, N) = 1.

**Theorem**: `OrderFinding.v:lagrange_bound` -- coprimality precondition.

**Fix**: Choose a different base `a` that is coprime to `N`.

---

## Arithmetic errors

### Overflow

```
integer overflow in {operation}
```

**Cause**: A u64 or u128 computation overflowed. Coq's `nat` type is unbounded, but Rust's fixed-width integers are not.

**Fix**: This indicates a need for wider arithmetic. The `operation` field names the specific computation (e.g., "multiply", "scale_and_round"). Report this as a bug if it occurs with standard configurations.

### InvalidParameter

```
invalid parameter: {message}
```

**Cause**: A parameter value is outside its valid range. The `message` field contains specifics.

**Fix**: Check the message for which parameter is invalid and correct it.

---

## Cryptographic errors

### DecryptionFailed

```
decryption failed: noise exceeded budget
```

**Cause**: The ciphertext's noise has grown beyond the decryption threshold. The plaintext cannot be recovered.

**Fix**: This is the consequence of ignoring `NoiseOverflow` or `NoiseBudgetExhausted` warnings. Reduce circuit depth, add bootstrap calls, or use a larger modulus configuration.

### KeyGenFailed

```
key generation failed: {reason}
```

**Cause**: Key generation encountered an error. The `reason` field contains specifics (e.g., NTT initialization failure, insufficient entropy).

**Fix**: Check the reason. Most commonly caused by invalid config parameters (N not a power of 2, q not NTT-friendly).

### SecurityLevelNotMet

```
security level not met: {bits} bits < required {required} bits
```

**Cause**: The configuration's security level (in bits) falls below the required minimum.

**Fix**: Use a `SecureConfig` preset (`secure_128`, `secure_192`, `secure_256`) rather than custom parameters. If using custom parameters, increase N or decrease log2(q).

---

## Encoding errors

### MessageOutOfBounds

```
message out of bounds: {message} >= plaintext modulus {modulus}
```

**Cause**: The plaintext value to encrypt is >= t (the plaintext modulus). BFV encoding requires `m < t`.

**Fix**: Reduce the plaintext value to the range [0, t). For `secure_128`, t=2053.

### InvalidPolynomialDegree

```
invalid polynomial degree: got {got}, expected {expected}
```

**Cause**: A polynomial was provided with a degree that does not match the ring dimension N.

**Fix**: Ensure all polynomials have exactly N coefficients.

### KeyRegimeMismatch

```
key regime mismatch: {message}
```

**Cause**: A key was used with an incompatible ciphertext or context. For example, using a Single-RNS key with a Dual-RNS ciphertext.

**Fix**: Ensure keys and ciphertexts were generated from the same context. Do not mix Single and Dual RNS regimes.

---

## Configuration errors

### NTTConfigError

```
NTT configuration error: {message}
```

**Cause**: The NTT engine could not be initialized with the given parameters. Common reasons: q is not NTT-friendly (does not have a primitive 2N-th root of unity), or N is not a power of 2.

**Fix**: Use a known NTT-friendly prime (e.g., 998244353) or a `SecureConfig` preset.

### ConfigError

```
configuration error: {message}
```

**Cause**: General configuration validation failure. The `message` field provides details.

**Fix**: Check the message. Common causes: N=0, t=0, empty modulus chain, q < t.

---

## Batching errors

### BatchingNotSupported

```
batching not supported for t={t}, N={n}: {reason}
```

**Cause**: CRT batching requires `t = 1 (mod 2N)` for the cyclotomic polynomial to split into N linear factors over Z_t. If this condition is not met, batching cannot encode multiple values into a single ciphertext.

**Fix**: Choose a plaintext modulus t that satisfies `t mod (2*N) == 1`. The `SecureConfig` presets are chosen to support batching.

### TooManySlotValues

```
too many slot values: got {got}, max {max}
```

**Cause**: More values were provided to the batch encoder than the ring can hold. The maximum number of slots is N (the polynomial degree).

**Fix**: Reduce the number of values or increase N.

### NoModularInverse

```
no modular inverse: {value}^(-1) mod {modulus} does not exist
```

**Cause**: A modular inverse was needed but does not exist because the value and modulus share a common factor.

**Fix**: This typically indicates an issue with the batching root computation. Check that t is prime or that the specific slot index is valid.

---

## Serialization errors

### DeserializationError

```
deserialization error: {message}
```

**Cause**: Deserializing untrusted input failed validation. This is a security boundary: malformed data is rejected to prevent DoS.

**Fix**: Verify the data source. If deserializing ciphertexts from an untrusted party, this error is expected for malformed input.

---

## Bootstrap errors

### BootstrapFailed

```
bootstrap failed: {reason}
```

**Cause**: The bootstrap procedure did not complete successfully. The `reason` field provides specifics.

**Recoverable**: Yes. May be retried with different parameters.

**Fix**: Check the reason. Common causes: insufficient boot key, config mismatch between boot and work contexts.

### BootstrapConfigMismatch

```
bootstrap config mismatch: {reason}
```

**Cause**: The bootstrap configuration does not match the working configuration. For example, the boot secret key was generated for a different ring dimension.

**Fix**: Regenerate bootstrap keys from the same configuration used for the working context.

### BootstrapOverflow

```
bootstrap arithmetic overflow in: {operation}
```

**Cause**: An intermediate value during bootstrap exceeded the representable range.

**Fix**: Report as a bug with the specific operation name. This should not occur with standard configurations.

---

## Entropy errors

### EntropyFailure

```
entropy source failure: {reason}
```

**Cause**: The OS cryptographic random number generator failed to provide entropy. This indicates a fundamental system-level problem (e.g., `/dev/urandom` unavailable, container without entropy source).

**Fix**: Ensure the system has a working CSPRNG. On Linux, check that `/dev/urandom` is accessible. On containers, ensure the entropy source is mounted.

---

## Noise Budget errors

### NoiseBudgetExhausted

```
noise budget exhausted: needed {required_mb} millibits, had {available_mb}
```

**Cause**: The operation would consume more noise budget than is available. Proceeding would make decryption unreliable.

**Recoverable**: Yes. Bootstrap to refresh the noise budget.

**Security note**: This error MUST be returned (never silently swallowed). IBM's 2025 BFV key recovery attack exploits noise overflow to create a decryption failure oracle. Silently allowing noise-exceeded ciphertexts through creates an exploitable vulnerability.

**Fix**: Bootstrap before the noise budget runs out. Use `AutoBootstrapEvaluator` for automatic management, or check `NoiseBudget::remaining_millibits()` before operations.

---

## Regime errors

### RegimeMismatch

```
regime mismatch in {operation}: expected {expected}, got {got}
```

**Cause**: A ciphertext, key, or operation does not match the expected RNS regime. For example, attempting a Dual K-Elimination operation on a Single-RNS ciphertext, or vice versa.

**Fix**: Ensure consistency: if you created a context with `RNSFHEContext::new()` (Dual regime), use `encrypt_dual` / `decrypt_dual` / `mul_dual_*`. Do not mix Single and Dual operations on the same ciphertext.

---

## Error categories summary

| Category | Variants | Recoverable? |
|----------|----------|-------------|
| K-Elimination | NotCoprime, RangeOverflow, ModulusZero, AnchorZero, InexactDivision | No |
| GSO-FHE | NoiseOverflow, DepthExceeded | Yes (bootstrap) |
| Order Finding | OrderNotFound, NotCoprimeToModulus | No |
| Arithmetic | Overflow, InvalidParameter | No |
| Cryptographic | DecryptionFailed, KeyGenFailed, SecurityLevelNotMet | No |
| Encoding | MessageOutOfBounds, InvalidPolynomialDegree, KeyRegimeMismatch | No |
| Configuration | NTTConfigError, ConfigError | No |
| Batching | BatchingNotSupported, TooManySlotValues, NoModularInverse | No |
| Serialization | DeserializationError | No |
| Bootstrap | BootstrapFailed, BootstrapConfigMismatch, BootstrapOverflow | Partially |
| Entropy | EntropyFailure | No |
| Noise | NoiseBudgetExhausted | Yes (bootstrap) |
| Regime | RegimeMismatch | No |

---

## Where to go next

- [Troubleshooting](troubleshooting) -- common failure scenarios and how to resolve them.
- [Noise Budget](noise-budget) -- understanding the noise tracking system that produces NoiseOverflow and NoiseBudgetExhausted.
- [Security Configs](security-configs) -- choosing parameters that avoid SecurityLevelNotMet.
