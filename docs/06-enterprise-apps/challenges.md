# Track 6 — Enterprise Apps: Challenges

These are capstone challenges. Each one requires skills from multiple tracks. They are the kind of projects you would put in a portfolio.

---

## Challenge 1 — Full-stack E-commerce Backend

Build a production-ready e-commerce API:

**Features:**
- User registration/login with JWT + refresh tokens (hashed passwords with `argon2`).
- Products: CRUD, categories, search by name, sort by price.
- Shopping cart: add/remove/update items (stored in DB per session).
- Orders: create from cart, deduct stock in a transaction, order history.
- Async order confirmation email via a Kafka event (`orders.confirmed` topic).

**Requirements:**
- PostgreSQL with SQLx and migrations.
- Structured JSON logging with `tracing`.
- OpenAPI spec at `/swagger-ui`.
- Docker + docker-compose for local dev.
- 80%+ test coverage (`cargo test`).

---

## Challenge 2 — Real-time Analytics Pipeline

Build an event ingestion and real-time analytics service:

1. `POST /events` — ingest JSON events (`{"type": "page_view", "user_id": "...", "url": "..."}`) at high throughput. Respond immediately with `202 Accepted`.
2. A background worker processes events from an in-memory queue and aggregates: page views per URL per minute, unique users per hour.
3. `GET /analytics/pages` — returns top 20 pages by views (last 24h).
4. `GET /analytics/users` — returns hourly unique user counts.
5. `GET /metrics` — exposes Prometheus metrics: events ingested/sec, queue depth, processing latency.

Target: handle 10,000 events/second on a single machine. Benchmark with `wrk` or `oha`.

---

## Challenge 3 — Microservice Mesh

Build three Rust microservices that communicate via NATS:

| Service | Responsibility |
|---------|----------------|
| `users-svc` | User accounts, auth, JWT issuance |
| `products-svc` | Product catalogue, inventory |
| `orders-svc` | Order processing, publishes events |

Requirements:
- Each service has its own PostgreSQL database.
- `orders-svc` publishes to NATS topic `orders.created` when an order is placed.
- `products-svc` subscribes to `orders.created` and decrements inventory.
- An API gateway (Axum) on port 3000 routes traffic to the three services.
- Each service has structured JSON logs with a `service` field.
- Docker Compose file that starts all services + their DBs + NATS.

---

## Challenge 4 — WASM Plugin System

Build an Axum web service with a WebAssembly plugin architecture:

1. Define a plugin interface: WASM modules export `transform(input: &str) -> String`.
2. The host service loads `.wasm` files from a `./plugins/` directory at startup using `wasmtime`.
3. `POST /transform/:plugin_name` runs the named plugin on the request body and returns the result.
4. Implement 3 plugins in Rust (compiled to WASM): `uppercase`, `word_count` (returns JSON), `markdown_to_html` (use the `pulldown-cmark` crate compiled to WASM).
5. Plugins can be hot-reloaded: `POST /admin/reload` rescans the plugins directory.

**Crates:** `wasmtime`, `pulldown-cmark`.

---

## Challenge 5 — AI-assisted Code Analysis Service

Build a complete code quality service combining all tracks:

1. **REST API** (Axum): `POST /analyse` accepts Rust source code.
2. **Static analysis**: run `cargo clippy` via `std::process::Command` in a sandboxed temp directory, parse the output.
3. **AI review**: send the code to an LLM (OpenAI / watsonx) with a code review system prompt.
4. **Persistence**: store all analyses in PostgreSQL (code hash, clippy output, AI feedback, timestamp).
5. **Deduplication**: if the same code (by SHA-256 hash) was analysed within 24h, return the cached result.
6. **Events**: publish an `analysis.completed` event to NATS.
7. **Observability**: OpenTelemetry traces, Prometheus metrics, structured logs.
8. **Docker**: Dockerfile + docker-compose with the service, PostgreSQL, and NATS.

---

[Back to Practice](practice.md){ .md-button }

[Back to the beginning](../index.md){ .md-button .md-button--primary }
