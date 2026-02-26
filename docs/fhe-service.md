---
layout: default
title: FHE Service
parent: Using the System
nav_order: 8
---

# FHE Service

[Home](index)

HTTP microservice for session-based FHE operations. Key material stays on the server; only ciphertexts travel over the wire.

Source files: `crates/fhe-service/src/main.rs`, `session.rs`, `handlers.rs`, `wire.rs`, `http.rs`.

---

## What It Does

The FHE service wraps `RNSFHEContext` in a stateful HTTP server that manages per-client sessions. Each session holds:

- An `RNSFHEContext` (DualRNS with 5-anchor K-Elimination)
- A `DualRNSFullKeySet` (secret + public + evaluation keys)
- A `NoiseBudget` tracker
- An operation counter and timestamps

The secret key **never leaves the server**. Clients send plaintext values for encryption and receive base64-encoded ciphertexts. Homomorphic operations are performed server-side.

---

## REST API

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/healthz` | Health check (returns active session count) |
| GET | `/v1/version` | Service version |
| GET | `/v1/metrics` | Request counts, uptime, session stats |
| POST | `/v1/sessions` | Create a new FHE session |
| GET | `/v1/sessions/{id}` | Get session info (config, noise budget, op count) |
| DELETE | `/v1/sessions/{id}` | Destroy a session (zeros key material) |
| POST | `/v1/sessions/{id}/encrypt` | Encrypt plaintext values |
| POST | `/v1/sessions/{id}/decrypt` | Decrypt ciphertexts |
| POST | `/v1/sessions/{id}/evaluate` | Homomorphic operations |

### Create Session

```json
POST /v1/sessions
{
    "config": "secure_128"
}
```

Supported configs: `secure_128`, `secure_192`, `secure_256`.

Response:

```json
{
    "session_id": "sess_a1b2c3d4...",
    "config": "secure_128",
    "params": {
        "n": 4096,
        "log_q": 90,
        "t": 65537,
        "security_bits": 128
    },
    "noise_budget_estimate_millibits": 30000
}
```

Session IDs are generated from 16 bytes of OS CSPRNG entropy, hex-encoded with a `sess_` prefix.

### Encrypt

```json
POST /v1/sessions/{id}/encrypt
{
    "values": [42, 7, 100]
}
```

Response:

```json
{
    "ciphertexts": ["<base64 bincode>", "..."],
    "noise_budget_estimate_millibits": 29500
}
```

Each value must be `< t`. Ciphertexts are serialized as bincode, then base64-encoded.

### Decrypt

```json
POST /v1/sessions/{id}/decrypt
{
    "ciphertexts": ["<base64 bincode>", "..."]
}
```

Response:

```json
{
    "values": [42, 7, 100],
    "noise_budget_estimate_millibits": 29500
}
```

### Evaluate (Homomorphic Operations)

```json
POST /v1/sessions/{id}/evaluate
{
    "operations": [
        {
            "op": "add",
            "inputs": ["<ct_a base64>", "<ct_b base64>"]
        },
        {
            "op": "mul_plain",
            "inputs": ["<ct base64>"],
            "scalar": 5
        }
    ]
}
```

Supported operations:
- `add`: ct + ct
- `sub`: ct - ct
- `negate`: -ct
- `add_plain`: ct + scalar
- `mul_plain`: ct * scalar
- `mul`: ct * ct (requires eval key)

---

## Session Management

### SessionStore

Thread-safe session storage using `Arc<RwLock<HashMap>>`.

```rust
pub struct SessionStore {
    sessions: Arc<RwLock<HashMap<String, Session>>>,
    max_sessions: usize,
    ttl_seconds: u64,
}
```

Configurable via environment variables:
- `FHE_MAX_SESSIONS`: maximum concurrent sessions (default: 64)
- `FHE_SESSION_TTL`: session time-to-live in seconds

### Session

Each session holds a complete DualRNS FHE context:

```rust
pub struct Session {
    pub session_id: String,
    pub config_name: String,
    pub config: FHEConfig,
    pub noise_budget: NoiseBudget,
    pub operation_count: u64,
    pub created_at: u64,
    pub last_accessed: u64,
    pub rns_ctx: RNSFHEContext,
    pub dual_keys: DualRNSFullKeySet,
}
```

Session creation uses OS CSPRNG for all key generation:

```rust
let session = Session::new("secure_128")?;
// Internally calls:
//   RNSFHEContext::try_new(&config)
//   rns_ctx.generate_keys_dual_full_secure()
```

---

## Wire Types

Defined in `wire.rs`. All request/response types use serde for JSON serialization.

### Size Limits (DoS Protection)

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAX_STRING_FIELD_LEN` | 1 KB | Non-ciphertext string fields |
| `MAX_CIPHERTEXT_FIELD_LEN` | 4 MB | Single ciphertext (N=16384 DualRNS) |
| `MAX_REQUEST_ALLOCATION` | 64 MB | Total per-request memory cap |

---

## Serialization

Ciphertexts are serialized using bincode (compact binary format) and base64-encoded for JSON transport. The `DualRNSCiphertext` type supports serde when the `serde` feature is enabled on the `nine65` crate.

The service validates all deserialized ciphertexts before use:
- Polynomial degree matches expected N
- Number of RNS limbs matches expected prime count
- All coefficients are within bounds

---

## Cloud Run Deployment

| Parameter | Value |
|-----------|-------|
| Platform | Google Cloud Run |
| Service name | `nine65-v7` |
| Region | `us-south1` (Dallas) |
| Project | `astro-resonance` |
| Container port | 8080 |
| URL | `https://nine65-v7-517338038154.us-south1.run.app` |
| Status | **Disabled** (billing paused) |
| Deploy method | Push to main triggers Cloud Build |

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `FHE_SERVICE_HOST` | `127.0.0.1` | Bind address |
| `FHE_SERVICE_PORT` | `8080` | Bind port |
| `FHE_MAX_SESSIONS` | `64` | Maximum concurrent sessions |
| `FHE_MAX_CONNECTIONS` | `256` | Maximum concurrent TCP connections |
| `FHE_SESSION_TTL` | (none) | Session TTL in seconds |

### Security

- `#![deny(clippy::float_arithmetic)]` enforced at crate level
- `allow_insecure` feature blocked in release builds via `compile_error!`
- Session IDs generated from OS CSPRNG (16 random bytes)
- All key generation uses `generate_keys_dual_full_secure()` (OS CSPRNG)
- Ciphertext validation on every deserialization (prevents DoS via malformed inputs)
- Connection limiting prevents resource exhaustion

---

## Running Locally

```bash
cd crates/fhe-service
cargo run --release

# Custom config
FHE_SERVICE_PORT=9090 FHE_MAX_SESSIONS=128 cargo run --release
```

### Testing

```bash
# Create a session
curl -X POST http://localhost:8080/v1/sessions \
  -H "Content-Type: application/json" \
  -d '{"config": "secure_128"}'

# Health check
curl http://localhost:8080/healthz
```

---

## Feature Flag Requirements

The `fhe-service` crate requires the `serde` feature on `nine65` for ciphertext serialization:

```toml
[dependencies]
nine65 = { path = "../nine65", features = ["serde", "ntt_fft"] }
```

Without the `serde` feature, ciphertexts cannot be serialized for network transport.

---

## Where to go next

- [Cookbook](cookbook) -- code recipes for the operations the service wraps
- [Key Management](key-management) -- how session keys are generated and protected
- [Security Configs](security-configs) -- the `secure_128`/`secure_192`/`secure_256` parameters
