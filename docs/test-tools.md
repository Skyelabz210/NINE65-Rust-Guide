---
layout: default
title: Test Tools
parent: Tools & Testing
nav_order: 3
---

# Test Tools


Beyond the built-in `cargo test`, these tools give deeper visibility into test runs.

## cargo-nextest

Drop-in replacement for `cargo test` with better output.

### What it gives you
- Per-test timing on stable (no nightly needed)
- Live progress bar showing which tests are running
- Retry support for flaky tests
- JUnit XML output for CI or browser viewing
- Better failure diagnostics

### Install

```bash
cargo install cargo-nextest
```

### Usage

```bash
# Run all core tests (replaces cargo test -p nine65 --lib --release)
cargo nextest run -p nine65 --lib --release

# With filter
cargo nextest run -p nine65 --lib --release -- bootstrap

# List tests
cargo nextest list -p nine65 --lib --release

# Generate JUnit XML report
cargo nextest run -p nine65 --lib --release --message-format libtest-json > results.xml
```

## cargo-pretty-test

Renders test results as a module tree with colors and timing.

### Install

```bash
cargo install cargo-pretty-test
```

### Usage

```bash
cargo pretty-test -p nine65 --lib --release
```

## cargo-llvm-cov

Code coverage --- shows which lines of code your tests actually execute.

### What it gives you
- HTML report you open in a browser
- Every file with green (covered) / red (not covered) line highlighting
- Summary showing percentage coverage per module
- Identifies dead code and untested paths

### Install

```bash
cargo install cargo-llvm-cov
```

You also need the llvm-tools component:

```bash
rustup component add llvm-tools-preview
```

### Usage

```bash
# Generate HTML coverage report
cargo llvm-cov --release --workspace --exclude nine65-python --exclude nine65-wasm --html

# Open the report
# Output goes to target/llvm-cov/html/index.html
xdg-open target/llvm-cov/html/index.html

# Just core library coverage
cargo llvm-cov -p nine65 --lib --release --html

# Text summary (no HTML)
cargo llvm-cov -p nine65 --lib --release

# JSON output for tooling
cargo llvm-cov -p nine65 --lib --release --json
```

### Reading the Report

The HTML report shows:
- **Green lines**: Executed by at least one test
- **Red lines**: Never executed --- potential gaps
- **Per-file percentage**: How much of each file is covered
- **Per-function coverage**: Which functions were tested

Focus on red lines in critical files:
- `ops/bootstrap.rs` --- every bootstrap path should be green
- `arithmetic/k_elimination.rs` --- CRT reconstruction must be covered
- `noise/budget.rs` --- all budget operations should be tested
- `security/` --- constant-time operations need full coverage

## Miri (Nightly Only)

Memory safety checker. Detects undefined behavior that normal tests can't catch.

### What it catches
- Use-after-free
- Out-of-bounds access
- Uninitialized memory reads
- Data races
- Invalid pointer arithmetic

### Install

```bash
rustup toolchain install nightly
rustup +nightly component add miri
```

### Usage

```bash
# Run a specific test under miri (very slow)
cargo +nightly miri test -p nine65 --lib -- montgomery::tests::test_montgomery_mul

# Run with more diagnostics
MIRIFLAGS="-Zmiri-backtrace=full" cargo +nightly miri test -p nine65 --lib -- specific_test
```

Miri is 100-1000x slower than normal test execution. Use it for targeted verification of unsafe code or security-sensitive arithmetic, not for full suite runs.

### Limitations
- Cannot handle FFI calls
- Cannot handle inline assembly
- Some system calls are unsupported
- Very slow --- minutes per test

## Comparison

| Tool | Speed | What it shows | When to use |
|------|-------|-------------|-------------|
| `cargo test` | Fast | Pass/fail | Daily workflow |
| `cargo test -- --nocapture` | Fast | Pass/fail + println output | Debugging specific behavior |
| `cargo nextest run` | Fast | Pass/fail + timing + progress | When you want timing data |
| `cargo pretty-test` | Fast | Module tree view | Visual overview |
| `cargo llvm-cov` | Medium | Line-level coverage | Finding untested code |
| `cargo +nightly miri test` | Very slow | Memory safety violations | Verifying unsafe code |
