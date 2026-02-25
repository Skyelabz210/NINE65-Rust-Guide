---
layout: default
title: Noise Budget
---

# Noise Budget

[Home](index)

## What Noise Is

Every ciphertext carries "noise" --- random values added during encryption that hide the plaintext. Each operation (especially multiplication) increases this noise. When noise exceeds the budget, decryption returns garbage instead of the correct plaintext.

## Millibits

NINE65 tracks noise in **millibits** (1/1000 of a bit). This avoids floating-point arithmetic while maintaining fine-grained precision.

Example: a budget of 62,000 millibits = 62 bits of noise headroom.

## Budget Lifecycle

```
Fresh ciphertext  ──>  Operations  ──>  Budget exhausted  ──>  Bootstrap  ──>  Fresh again
   62,000 mb              -43,000 mb         ~0 mb              reset          62,000 mb
```

### Costs per operation (secure_128)

| Operation | Cost (millibits) | Notes |
|-----------|-----------------|-------|
| Addition | ~100 | Cheap --- rarely an issue |
| Multiplication | ~31,000 | The expensive one |
| Relinearization | ~12,000 | Applied after every multiplication |
| **Total mul cycle** | **~43,000** | mul + relin combined |
| Rescaling | Negative (refund) | Rescaling reduces noise |

### Budget for secure_128

- Initial budget: 62,000 mb
- After 1 mul: 62,000 - 43,000 = 19,000 mb remaining
- After 2 muls: 19,000 - 43,000 = negative --- budget exhausted
- Max depth without bootstrap: **2 multiplications**

## Key Functions

### NoiseBudget (noise/budget.rs)

```rust
// Create from config
let budget = NoiseBudget::from_config(&config);

// Check remaining
let remaining: i64 = budget.remaining_millibits();

// Consume budget for an operation
// Returns Ok(()) if budget sufficient, Err if exhausted
budget.consume(NoiseOpType::MulCt, cost_mb)?;

// Check if bootstrap should trigger
// trigger_permille: 250 = trigger at 25% remaining
let should_boot: bool = budget.should_bootstrap(250);

// Reset after bootstrap
budget.reset_after_bootstrap(&config);

// Can this operation be performed?
let can: bool = budget.can_perform(NoiseOpType::MulCt, cost_mb);
```

### Important: consume() error handling

`consume()` returns `Err` when the budget is insufficient. It does NOT decrement the remaining budget in that case --- it stays at whatever value it had before.

This caused the auto-bootstrap bug: the code did `let _ = budget.consume(...)` (discarding the error), so when budget was exhausted, `remaining_mb` froze above the trigger threshold, and `should_bootstrap()` never returned true.

The fix: check the return value. If `Err`, the budget IS exhausted --- bootstrap unconditionally.

### can_perform() with negative costs

`can_perform()` returns `true` unconditionally when the cost is negative (rescaling/noise-reducing operations). This was Fix 3 from the audit --- rescaling should always be allowed because it improves the budget.

## Auto-Bootstrap Threshold

The `AutoBootstrapEvaluator` triggers bootstrap when:

1. `budget.consume()` returns `Err` (budget exhausted), OR
2. `budget.should_bootstrap(trigger_permille)` returns true (budget below threshold)

Default threshold: 250 permille (25% remaining). Configurable via `set_trigger_threshold()`.

## Viewing Budget in Tests

Tests that print noise information use `--nocapture`:

```bash
cargo test -p nine65-extreme-tests --features extreme-tests --release -- depth_stress --nocapture
```

The depth ceiling tests print noise state at each multiplication depth.
