# crypto-cex

A lightweight cryptocurrency exchange backend built with Rust and [Actix Web](https://actix.rs/). It provides user registration, JWT-based authentication, USD balance management, and asset deposit tracking — all backed by in-memory state suitable for learning and prototyping.

## Features

- **User accounts** — Sign up and sign in with username/password
- **JWT authentication** — Protected routes require a valid bearer token
- **USD balances** — Managed on a dedicated background thread via message passing
- **On-ramp** — Add USD to a user's balance
- **Asset deposits** — Track per-user holdings by asset symbol (e.g. `BTC`, `ETH`)
- **Balance lookup** — Fetch USD balance and all stock/asset balances for the authenticated user

## Tech Stack

| Layer | Technology |
|-------|------------|
| Web framework | Actix Web 4 |
| Serialization | Serde / Serde JSON |
| Auth | JSON Web Tokens (`jsonwebtoken`) |
| Concurrency | `std::sync::Mutex`, `mpsc` channels, background thread |

## Project Structure

```
rust/
├── Cargo.toml
├── src/
│   ├── main.rs              # Server entry point, app state, balance worker thread
│   ├── middleware/
│   │   └── mod.rs           # JWT auth extractor (`AuthUser`)
│   ├── routes/
│   │   ├── mod.rs
│   │   └── user.rs          # HTTP handlers (signup, signin, balance, onramp, deposit)
│   └── types/
│       ├── mod.rs
│       └── user.rs          # Request/response types and domain models
└── README.md
```

## Prerequisites

- [Rust](https://www.rust-lang.org/tools/install) (2024 edition)
- `curl` or any HTTP client for testing

## Getting Started

### 1. Clone and enter the project

```bash
cd rust
```

### 2. Install dependencies

Dependencies are managed by Cargo and resolved automatically on build:

```bash
cargo build
```

### 3. Run the server

```bash
cargo run
```

The server listens on **`http://127.0.0.1:3001`**.

> **Note:** All data is stored in memory. Restarting the server clears users, balances, and deposits.

## Authentication

Protected endpoints require a JWT in the `Authorization` header.

```
Authorization: Bearer <token>
```

Tokens are issued by `POST /signin` and expire after **24 hours**. The signing secret is currently hardcoded as `"secret"` (see `src/middleware/mod.rs`) — replace this before any production use.

## API Reference

### Public endpoints

#### `POST /signup`

Create a new user account.

**Request body**

```json
{
  "username": "alice",
  "password": "secret123"
}
```

**Responses**

| Status | Body |
|--------|------|
| `200 OK` | `{ "message": "Successfully signed up" }` |
| `401 Unauthorized` | `{ "message": "User already" }` |

**Example**

```bash
curl -X POST http://127.0.0.1:3001/signup \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"secret123"}'
```

---

#### `POST /signin`

Authenticate and receive a JWT.

**Request body**

```json
{
  "username": "alice",
  "password": "secret123"
}
```

**Response (`200 OK`)**

```json
{
  "token": "<jwt>"
}
```

**Example**

```bash
curl -X POST http://127.0.0.1:3001/signin \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"secret123"}'
```

---

### Protected endpoints

All routes below require:

```
Authorization: Bearer <token>
```

#### `GET /balance`

Return the authenticated user's USD balance and asset holdings.

**Response (`200 OK`)**

```json
{
  "usdBalance": 1000,
  "stockBalances": {
    "BTC": 2,
    "ETH": 5
  }
}
```

**Example**

```bash
curl http://127.0.0.1:3001/balance \
  -H "Authorization: Bearer <token>"
```

---

#### `POST /onramp`

Add USD to the authenticated user's balance.

**Request body**

```json
{
  "qty": 500
}
```

**Response:** `200 OK` (empty body)

**Example**

```bash
curl -X POST http://127.0.0.1:3001/onramp \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"qty":500}'
```

---

#### `POST /deposit/{asset_symbol}`

Deposit units of a given asset into the user's portfolio.

**Path parameter:** `asset_symbol` — e.g. `BTC`, `ETH`

**Request body**

```json
{
  "qty": 2
}
```

**Response (`200 OK`)**

```json
{
  "message": "Successfully deposited"
}
```

**Example**

```bash
curl -X POST http://127.0.0.1:3001/deposit/BTC \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"qty":2}'
```

---

### Planned endpoints

The following handlers exist in `src/routes/user.rs` but are **not yet registered** on the server:

| Method | Path | Status |
|--------|------|--------|
| `POST` | `/order` | Stub — returns `200 OK` |
| `POST` | `/cancel` | Stub — returns `200 OK` |

## Architecture

```
┌─────────────┐     HTTP      ┌──────────────────┐
│   Client    │ ────────────► │   Actix Web      │
└─────────────┘               │   (main thread)  │
                              └────────┬─────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
              ┌─────▼─────┐    ┌───────▼───────┐   ┌──────▼──────┐
              │  users    │    │ stock_balances│   │ balances_tx │
              │  (Mutex)  │    │   (Mutex)     │   │  (mpsc)     │
              └───────────┘    └───────────────┘   └──────┬──────┘
                                                            │
                                              ┌─────────────▼─────────────┐
                                              │  Balance worker thread    │
                                              │  (USD balances HashMap)   │
                                              └───────────────────────────┘
```

- **Users and stock balances** live in `Mutex`-protected structures on the Actix runtime.
- **USD balances** are updated asynchronously through an `mpsc` channel consumed by a dedicated background thread, keeping balance mutations off the request hot path.
- **JWT validation** is handled by the `AuthUser` extractor in middleware, which decodes the token and exposes the user ID to handlers.

## Development

### Run in debug mode

```bash
cargo run
```

### Run tests

```bash
cargo test
```

### Build for release

```bash
cargo build --release
```

The release binary is written to `target/release/rust`.

## Limitations

This is a **prototype / learning project**, not production-ready software:

- Passwords are stored in plain text
- JWT secret is hardcoded
- All state is in-memory and lost on restart
- No input validation, rate limiting, or HTTPS
- Order matching and cancellation are not implemented

## License

Not specified.
