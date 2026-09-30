# Track 6 — Enterprise Apps: Practice

---

## Exercise 1 — SQLx CRUD with PostgreSQL

Build a `products` service backed by PostgreSQL:

1. Write a migration that creates a `products` table: `id SERIAL PRIMARY KEY, name VARCHAR, price NUMERIC, stock INT, created_at TIMESTAMPTZ`.
2. Implement CRUD operations using `sqlx::query_as!` macros.
3. Run migrations at startup with `sqlx::migrate!`.
4. Expose as an Axum REST API (full CRUD, pagination on list endpoint).

**Local setup:**
```bash
docker run -d -e POSTGRES_PASSWORD=dev -p 5432:5432 postgres:16-alpine
export DATABASE_URL=postgresql://postgres:dev@localhost/dev
sqlx database create
```

---

## Exercise 2 — Database Transactions

Extend Exercise 1 with a purchase endpoint:

`POST /purchase` with body `{"product_id": 1, "quantity": 3}`:
1. Check stock in a transaction.
2. Deduct from stock.
3. Create a `purchases` record.
4. Commit; if any step fails, rollback and return an error.

Write a test that verifies the transaction rolls back correctly on failure.

---

## Exercise 3 — Structured Logging

Add `tracing` to any previous project:

1. Initialise with JSON output format.
2. Add `#[instrument]` to at least 3 handler functions.
3. Log at `info` level: request received, DB query executed, response sent.
4. Log at `warn` level: slow queries (> 100ms), 404s.
5. Log at `error` level: DB errors, unexpected panics.

Verify JSON log lines appear on stdout with fields `timestamp`, `level`, `target`, `span`.

---

## Exercise 4 — Kafka Producer & Consumer

1. Run Kafka locally: `docker compose up` with the Bitnami Kafka image.
2. Write a **producer** binary that publishes 100 messages to topic `rust.events` (one per second).
3. Write a **consumer** binary that reads from `rust.events` and prints each message with offset and partition.
4. Run both concurrently and confirm all messages are received.

---

## Exercise 5 — JWT Middleware

Implement JWT authentication for the products API:

1. `POST /auth/token` accepts `{"username": "admin"}` and returns a signed JWT (1-hour expiry).
2. A middleware layer verifies the JWT on all `/products` routes.
3. The middleware extracts the `sub` claim and adds it to request extensions.
4. Handlers can access the current user via `Extension<CurrentUser>`.

---

## Exercise 6 — Criterion Benchmarks

Write benchmarks for:
1. Your `fizzbuzz` function from Track 1.
2. JSON serialisation of a `Vec<Product>` (10, 100, 1000 items).
3. A database SELECT query (use sqlx + test DB).

Run `cargo bench` and interpret the HTML report.

---

## Exercise 7 — WebAssembly Module

Compile a Rust library to WASM:

1. Create a `wasm_utils` library crate.
2. Export these functions with `wasm_bindgen`: `word_count(text: &str) -> u32`, `to_uppercase(s: &str) -> String`, `fibonacci(n: u32) -> u32`.
3. Build with `wasm-pack build --target web`.
4. Write a minimal HTML page that loads the WASM and calls all three functions.

---

[Go to Challenges](challenges.md){ .md-button .md-button--primary }

[Back to Learn](learn.md){ .md-button }
