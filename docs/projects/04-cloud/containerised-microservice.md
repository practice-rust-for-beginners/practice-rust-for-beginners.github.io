# Project: Containerised Microservice

**Track:** ☁️ Cloud &nbsp;|&nbsp; **Difficulty:** 🟡 Intermediate &nbsp;|&nbsp; **Crates:** axum, sqlx, tokio, dotenvy

---

## What You'll Build

Take a Rust API, give it a PostgreSQL database, containerise it with a multi-stage Dockerfile, and wire it all together with Docker Compose. The result is a production-pattern stack you can deploy anywhere.

```text
docker compose up --build
→ app  listening on :3000
→ db   PostgreSQL 16 on :5432

GET  /health          → {"status":"ok","db":"connected"}
POST /items           → create item (persisted to Postgres)
GET  /items           → list all items
```

---

## Tech Stack

```toml
[dependencies]
axum    = { version = "0.7", features = ["json"] }
sqlx    = { version = "0.8", features = ["runtime-tokio", "postgres", "macros", "chrono"] }
tokio   = { version = "1",   features = ["full"] }
serde   = { version = "1",   features = ["derive"] }
dotenvy = "0.15"
```

---

## Step 1 — Database Migration

```bash
cargo install sqlx-cli
export DATABASE_URL=postgresql://dev:dev@localhost/dev
sqlx database create
sqlx migrate add create_items
```

```sql
-- migrations/YYYYMMDDHHMMSS_create_items.sql
CREATE TABLE items (
    id         SERIAL PRIMARY KEY,
    name       VARCHAR(255) NOT NULL,
    quantity   INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

```bash
sqlx migrate run
```

---

## Step 2 — API with SQLx

```rust
// src/main.rs
use axum::{Router, routing::{get, post}, extract::State, Json, http::StatusCode};
use serde::{Deserialize, Serialize};
use sqlx::{PgPool, FromRow};
use std::env;

#[derive(Serialize, FromRow)]
struct Item { id: i32, name: String, quantity: i32 }

#[derive(Deserialize)]
struct CreateItem { name: String, quantity: i32 }

async fn health(State(pool): State<PgPool>) -> Json<serde_json::Value> {
    let db = sqlx::query_scalar::<_, i64>("SELECT 1")
        .fetch_one(&pool).await.is_ok();
    Json(serde_json::json!({
        "status": "ok",
        "db": if db { "connected" } else { "error" }
    }))
}

async fn list_items(State(pool): State<PgPool>) -> Json<Vec<Item>> {
    let items = sqlx::query_as!(Item, "SELECT id, name, quantity FROM items ORDER BY id")
        .fetch_all(&pool).await.unwrap_or_default();
    Json(items)
}

async fn create_item(State(pool): State<PgPool>, Json(b): Json<CreateItem>)
    -> (StatusCode, Json<Item>)
{
    let item = sqlx::query_as!(Item,
        "INSERT INTO items (name, quantity) VALUES ($1, $2) RETURNING id, name, quantity",
        b.name, b.quantity
    ).fetch_one(&pool).await.unwrap();
    (StatusCode::CREATED, Json(item))
}

#[tokio::main]
async fn main() {
    dotenvy::dotenv().ok();
    let database_url = env::var("DATABASE_URL").expect("DATABASE_URL must be set");
    let pool = PgPool::connect(&database_url).await.expect("DB connection failed");
    sqlx::migrate!().run(&pool).await.expect("Migrations failed");

    let app = Router::new()
        .route("/health", get(health))
        .route("/items",  get(list_items).post(create_item))
        .with_state(pool);

    let port = env::var("PORT").unwrap_or("3000".into());
    let listener = tokio::net::TcpListener::bind(format!("0.0.0.0:{port}")).await.unwrap();
    println!("Listening on :{port}");
    axum::serve(listener, app).await.unwrap();
}
```

---

## Step 3 — Multi-stage Dockerfile

```dockerfile
# Dockerfile
FROM rust:1.79-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo 'fn main(){}' > src/main.rs && cargo build --release
RUN rm -f target/release/deps/containerised*
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim AS runtime
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates libpq5 \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY --from=builder /app/target/release/containerised-microservice ./app
COPY migrations ./migrations
EXPOSE 3000
ENTRYPOINT ["./app"]
```

---

## Step 4 — Docker Compose

```yaml
# docker-compose.yml
version: "3.9"
services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://dev:dev@db:5432/dev
      - PORT=3000
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER:     dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB:       dev
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dev"]
      interval: 5s
      retries: 5
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

---

## Step 5 — Build & Run

```bash
docker compose up --build

# Test the API
curl http://localhost:3000/health
curl -X POST http://localhost:3000/items \
  -H "Content-Type: application/json" \
  -d '{"name":"Rust book","quantity":3}'
curl http://localhost:3000/items
```

---

## Stretch Goals

- [ ] Add `GET /items/:id` and `DELETE /items/:id`
- [ ] Add a `PATCH /items/:id` for partial updates
- [ ] Set up a nginx reverse proxy container in docker-compose
- [ ] Add a `docker-compose.prod.yml` with resource limits and no bind mounts

---

[Back to All Projects](../index.md){ .md-button }


[Next Project: Serverless URL Shortener](serverless-url-shortener.md){ .md-button .md-button--primary }
