---
layout: default
title: Boundary Proximity
parent: How It Works
nav_order: 11
---

# Boundary Proximity

[Home](index)

The boundary proximity system tracks how close values are to overflowing their integer representation limits. It provides early warning before K-Elimination capacity is exceeded or integer type promotions become necessary.

## The Problem

K-Elimination has a hard capacity boundary: values must be less than `alpha_cap * beta_cap` for exact reconstruction. If a value exceeds this boundary, K-Elimination silently returns an incorrect result. The boundary proximity system detects values approaching this limit before overflow occurs.

## Two Check Modes

### Approaching Boundary (80% / 90%)

Used before operations that may push values toward the anchor-product ceiling. This is the primary use case for FHE operations where tensor product intermediates grow toward the K-Elimination capacity.

### Post-Switch Margin (5% / 15%)

Used after an integer-type promotion (u64 to u128, or u128 to U256). Verifies that the promoted value has reasonable headroom in the new type. A value at 95%+ of the new type immediately after promotion suggests the anchor set is under-sized.

## CapacityRegion

The `CapacityRegion` enum categorizes a value's position relative to the capacity boundary:

| Region | Utilization | Meaning | Action |
|--------|------------|---------|--------|
| `Safe` | < 80% | Well within capacity | None needed |
| `Warn80` | 80--89% | Approaching boundary | Consider pre-emptive action |
| `Warn90` | 90--99% | Near overflow | Action strongly recommended |
| `Critical` | >= 100% | Overflow imminent or occurred | Must address immediately |

## CapacityReport

The result of a proximity check:

```rust
pub struct CapacityReport {
    pub region: CapacityRegion,
    pub utilization_pct: u8,     // 0--100+
    pub value_bits: u32,         // Bit-length of the value
    pub capacity_bits: u32,      // Bit-length of the capacity
}
```

Helper methods:
- `is_safe()` --- true if no warning is needed
- `is_warning()` --- true if any warning level is active
- `is_critical()` --- true if at or past the hard boundary
- `headroom_bits()` --- remaining bits before overflow

## Core Functions

### capacity_proximity_bits

```rust
pub fn capacity_proximity_bits(value_bits: u32, capacity_bits: u32) -> CapacityReport
```

Compares bit-lengths to determine the proximity region. Uses integer-only arithmetic: `utilization_pct = (value_bits * 100) / capacity_bits`. Accurate to approximately 2 bits at boundaries, which is sufficient for 80%/90% region detection.

Example:

```rust
let report = capacity_proximity_bits(127, 158);
// 127/158 = 80.4% -> Warn80
assert_eq!(report.region, CapacityRegion::Warn80);
```

### post_switch_margin_bits

```rust
pub fn post_switch_margin_bits(value_bits: u32, new_capacity_bits: u32) -> PostSwitchMargin
```

Returns a `PostSwitchMargin` indicating headroom after type promotion:

```rust
pub struct PostSwitchMargin {
    pub headroom_pct: u8,    // 100% - utilization%
    pub is_marginal: bool,   // 5--15% headroom
    pub is_critical: bool,   // < 5% headroom
}
```

Example:

```rust
// After u128 -> U256: value uses 130 bits, capacity is 256 bits
let margin = post_switch_margin_bits(130, 256);
// 49% headroom -- healthy
assert!(!margin.is_marginal);
assert!(!margin.is_critical);

// Tight case: value uses 248 bits after U256 promotion
let margin = post_switch_margin_bits(248, 256);
// 4% headroom -- critical
assert!(margin.is_critical);
```

### Bit-Length Helpers

```rust
pub fn u128_bit_length(value: u128) -> u32  // 128 - value.leading_zeros()
pub fn u64_bit_length(value: u64) -> u32    // 64 - value.leading_zeros()
```

These return 0 for value == 0 and the standard bit-length otherwise.

## Integration Points

### KElimination::capacity_proximity

```rust
let ke = KElimination::from_config(KElimConfig::Standard);
let report = ke.capacity_proximity(some_value);
if report.is_warning() {
    eprintln!(
        "K-Elimination at {}% capacity ({} of {} bits)",
        report.utilization_pct, report.value_bits, report.capacity_bits
    );
}
```

Checks a u128 value against this K-Elimination instance's `capacity_bits()`.

### DualRNSContext Methods

The `DualRNSContext` provides FHE-specific boundary checks:

| Method | Returns | Description |
|--------|---------|-------------|
| `anchor_capacity_bits()` | u32 | Actual anchor product bit-length (159 for 5 primes) |
| `anchor3_capacity_bits()` | u32 | What 3 anchors would give (~95 bits) --- for comparison |
| `check_k_proximity()` | CapacityReport | Check a value against anchor capacity |
| `check_intermediate_proximity()` | CapacityReport | Check tensor product intermediate |
| `max_intermediate_bits()` | u32 | Worst-case intermediate bit-length for current params |

### PyO3 and WASM Bindings

- **PyO3** (`nine65-python`): `FHEContext::boundary_report()` returns a `CapacityReport` to Python. `FHEContext::mul()` returns `PyResult<PyCiphertext>` with `catch_unwind` to surface capacity errors.
- **WASM** (`nine65-wasm`): `WasmFHEContext::boundary_report()` returns a string in the format `"region|pct|bits/cap"`. `WasmFHEContext::mul()` returns `Err(JsValue)` at 90% utilization.

## Production Safety

For all three production `SecureConfig` values, intermediate tensor product values sit at approximately 45--47% of anchor capacity:

| Config | Intermediate Bits | Anchor Capacity Bits | Utilization |
|--------|------------------|---------------------|-------------|
| secure_128 | ~71 | 159 | ~45% |
| secure_192 | ~73 | 159 | ~46% |
| secure_256 | ~75 | 159 | ~47% |

All values are firmly in the `Safe` region. The 5-anchor fix (commit 73a372e) ensured this by providing 159 bits of anchor capacity --- roughly double what the tensor product intermediates require.

## Integer-Only Implementation

The entire boundary system uses integer arithmetic exclusively. Bit-length comparisons serve as integer proxies for magnitude comparison. No f32 or f64 values appear anywhere in the module. The accuracy of approximately 2 bits at boundaries is more than sufficient for the 80%/90% threshold detection that the system performs.

## Where to go next

- [K-Elimination](k-elimination) --- the exact division algorithm whose capacity this system monitors
- [GSO-FHE](gso-fhe) --- the noise bounding system that uses boundary checks for noise estimation
- [BFV Parameters](bfv-params) --- the parameter choices that determine capacity requirements
