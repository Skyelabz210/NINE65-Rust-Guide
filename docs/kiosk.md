---
layout: default
title: Kiosk Architecture
parent: How It Works
nav_order: 12
---

# Kiosk Architecture

[Home](index)

The Kiosk system implements the "Ammunition" model: disposable, self-destructing computation units that run on client hardware and destroy themselves after use, leaving no forensic trace of the computation.

## The Ammunition Model

In NINE65's threat model, FHE computation units are like ammunition --- they are created for a specific purpose, used, and then destroyed. The Kiosk architecture provides three types of disposable units:

| Type | Uses | Lifetime | Use Case |
|------|------|----------|----------|
| **Bullet** | 1 | Single computation | One-shot encrypted query |
| **Capsule** | N | Budget-limited | Session-based encryption |
| **Fuse** | Unlimited | Time-limited | Time-boxed processing |

All unit types share the same destruction sequence and security properties. The difference is in their expiration trigger.

## KioskUnit

The `KioskUnit` struct wraps FHE state with self-destruction, integrity checking, and metering:

```rust
// Create a single-use Bullet
let mut unit = KioskUnit::bullet(id, &moduli, seed);

// Create a Capsule with 100-operation budget
let mut unit = KioskUnit::capsule(id, &moduli, seed, 100);

// Create a Fuse with 5-second lifetime
let mut unit = KioskUnit::fuse(id, &moduli, seed, 5_000_000_000);
```

### UnitStatus

Each unit progresses through a lifecycle:

| Status | Description |
|--------|-------------|
| `Ready` | Unit is created and waiting for data |
| `Active` | Data loaded, computation in progress |
| `Triggered` | Fuse blown or budget exhausted, destruction imminent |
| `Destroyed` | Self-destruction complete, receipt generated |
| `Compromised` | INV-8 check detected an attack, unit shut down |

### KioskLifecycle Trait

```rust
pub trait KioskLifecycle {
    fn should_destroy(&self) -> bool;
    fn destroy(&mut self) -> Option<DestructionReceipt>;
    fn status(&self) -> UnitStatus;
    fn unit_type(&self) -> KioskUnitType;
    fn unit_id(&self) -> u64;
}
```

### Checked Operations

Every arithmetic operation on a Kiosk unit is verified and metered:

```rust
// Load data into the unit
unit.load(&rns_limbs);

// Each operation checks INV-8 and consumes fuse entropy
let result = unit.checked_mul(limb_index, a, b, modulus);
// Returns None if unit is Triggered, Compromised, or budget exhausted
```

## FoldOperator (fold.rs)

The fold operator renders RNS state algebraically meaningless before zeroing, ensuring that no intermediate state can be recovered from memory forensics.

### Mechanism

1. **Fold**: Multiply each RNS limb by a pseudorandom mask derived from the Montgomery quotient stream. This destroys the algebraic relationship between limbs, making CRT reconstruction impossible.

2. **Unfold**: (During active computation only) Reverse the fold by dividing out the mask. Only possible while the fold key is in memory.

3. **Permanent Fold**: Fold + zeroize the fold key. After this, the original state is irrecoverable.

### FoldState

```rust
pub enum FoldState {
    Unfolded,           // Data in plaintext RNS form
    Folded,             // Data algebraically masked
    PermanentlyFolded,  // Mask key destroyed, data irrecoverable
    Zeroed,             // All state zeroed (terminal)
}
```

The fold uses Montgomery multiplication to apply masks, ensuring constant-time operation:

```rust
limbs[i] = mont.from_montgomery(
    mont.montgomery_mul(
        mont.to_montgomery(limbs[i]),
        mont.to_montgomery(masks[i]),
    )
);
```

The `FoldOperator` implements `Drop` to automatically zero all state when the operator goes out of scope.

## DestructionSequence (destruction.rs)

The destruction sequence is the core lifecycle event. It atomically performs:

1. **Permanent fold** the RNS limbs (algebraic obfuscation)
2. **Zero** all limb data via `zeroize`
3. **Zero** the fold operator state
4. **Generate** a SHA-256 receipt proving destruction

The entire sequence completes in under 20 microseconds --- far faster than a single FHE multiplication --- ensuring no observable window exists where intermediate state could be captured.

### DestructionReceipt

```rust
pub struct DestructionReceipt {
    pub hash: ReceiptHash,       // SHA-256 proof of destruction
    pub data: ReceiptData,       // Unit ID, operations count, timing, etc.
    pub duration_nanos: u64,     // How long the destruction took
}
```

Double destruction is prevented: calling `execute()` a second time returns `None`.

## EntropyFuse (fuse.rs)

The entropy fuse is a metering mechanism grounded in Montgomery multiplication. Each FHE operation consumes "entropy budget" derived from the Montgomery quotient stream.

### Why It Cannot Be Bypassed

The entropy budget is consumed by the computation itself, not by a software counter. Each Montgomery reduction produces a quotient `q_hat = (t * q_inv) mod R`. This quotient is algebraically irrecoverable (it was divided away). The fuse tracks cumulative entropy from these quotients. Skipping the measurement means skipping the Montgomery reduction, which means skipping the multiplication.

### FuseState

```rust
pub enum FuseState {
    Armed,      // Counting down
    Triggered,  // Budget exhausted, self-destruction imminent
    Spent,      // Unit destroyed
}
```

### Usage

```rust
// Create a fuse calibrated for 100 operations
let mut fuse = EntropyFuse::for_operations(100);

// Each operation feeds the Montgomery quotient
let state = fuse.consume(montgomery_quotient);
if state == FuseState::Triggered {
    // Self-destruction sequence begins
}
```

The fuse budget is tracked in **millibits** (1000 millibits = 1 bit of entropy) for fine-grained metering without floating-point.

## Inv8CheckLane (inv8.rs)

The INV-8 Check Lane detects algebraic inversion attacks (DDoS attacks that attempt to reverse-engineer the computation) at the first operation.

### Mechanism

For each FHE operation, the check lane:

1. Performs the same operation in a redundant check modulus (`p_check`)
2. Computes the expected result in the primary modulus
3. Verifies consistency between primary and check lanes

An algebraic inversion attack produces an inconsistent quotient stream, which the check lane detects immediately.

### Inv8Verdict

```rust
pub enum Inv8Verdict {
    Clean,              // No inversion signal
    InversionDetected,  // Attack detected
    Disabled,           // Check lane already flagged
}
```

The check primes are NTT-friendly primes independent of the main RNS moduli: 65537 (Fermat prime F4), 786433, 5767169, 23068673. The constructor selects a prime coprime to all main moduli.

The check lane operates in **strict mode** by default: `violation_threshold = 1`, meaning the first inconsistency immediately flags the unit as `Compromised`.

## ReceiptVerifier (receipt.rs)

Server-side receipt verification confirms that a Kiosk unit was properly destroyed without needing to observe the destruction:

```rust
let valid = ReceiptVerifier::verify(&receipt.data, &receipt.hash);
```

The `ReceiptData` covers:
- Unit ID (unique identifier)
- Fold generation count
- Final entropy level
- Timestamp (nanoseconds, monotonic clock)
- Operations performed count
- Unit type discriminant (Bullet=0, Capsule=1, Fuse=2)

The hash is SHA-256. Verification uses constant-time byte comparison to prevent timing side-channels.

Batch verification is supported:

```rust
let all_valid = ReceiptVerifier::verify_batch(&[(data1, hash1), (data2, hash2)]);
```

Batch verification uses bitwise AND (not short-circuit) to maintain constant-time behavior.

## Where to go next

- [Three-Lock Bootstrap](three-lock) --- the defense-in-depth system for protecting secrets during re-encryption
- [Montgomery Arithmetic](montgomery) --- the constant-time arithmetic underlying the fold and fuse mechanisms
- [Architecture](architecture) --- the high-level system design that Kiosk units plug into
