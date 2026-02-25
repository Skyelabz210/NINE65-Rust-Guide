---
layout: default
title: Home
---

# NINE65 Rust Guide

Personal reference for the NINE65 v7 FHE system. Everything needed to build, test, run, and understand the system.

## Quick Reference

| What | Command |
|------|---------|
| Build everything | `cargo build --release --workspace --exclude nine65-python --exclude nine65-wasm` |
| Run all 721 core tests | `cargo test -p nine65 --lib --release` |
| Run full 1,351 test suite | `cargo test --release --workspace --exclude nine65-python --exclude nine65-wasm` |
| Run with full output | append `-- --nocapture` |
| Run one area | append `-- bootstrap` (or `ntt`, `montgomery`, etc.) |
| Sanity check (30 sec) | `cargo test -p nine65 --lib --release -- test_auto_bootstrap_chained_muls` |

## Pages

- [Architecture](architecture) --- What NINE65 is and how it works
- [Building & Running](building) --- Build commands, binaries, configs
- [Testing](testing) --- Every test command, what they cover, verbose output
- [Test Tools](test-tools) --- nextest, pretty-test, llvm-cov, miri
- [Security Configs](security-configs) --- Parameters, what they mean, how to verify
- [Bootstrap System](bootstrap) --- Three paths, auto-bootstrap, depth ceilings
- [Crate Map](crate-map) --- What each crate does, key files
- [Noise Budget](noise-budget) --- How noise tracking works, millibits, thresholds
- [Extreme Tests](extreme-tests) --- The boundary/adversarial harness
- [Formal Proofs](formal-proofs) --- Coq and Lean4 verification
- [Troubleshooting](troubleshooting) --- Common errors and fixes
- [Rust Toolchain](rust-toolchain) --- Channels, components, proxy binaries
