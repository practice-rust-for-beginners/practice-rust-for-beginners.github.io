# Project: Microservice Mesh

**Track:** 🏢 Enterprise Apps &nbsp;|&nbsp; **Difficulty:** 🔴 Advanced &nbsp;|&nbsp; **Crates:** axum, sqlx, async-nats, tracing, tokio

---

## What You'll Build

Three independent Rust microservices communicating via NATS messaging, each with its own PostgreSQL database, behind a single API gateway. Every service has structured JSON logs and health endpoints.

```text
                    ┌─────────────┐
  HTTP :3000        │  API Gateway │
  ──────────────────┤  (gateway)   │
                    └──┬──┬──┬────┘
                       │  │  │
              ┌────────┘  │  └────────┐
              ▼           ▼           ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  users   │ │ products │ │  orders  │
        │  :3001   │ │  :3002   │ │  :3003   │
        │  db: pg1 │ │  db: pg2 │ │  db: pg3 │
        └──────────┘ └──────────┘ └──────────┘
               NATS pub/sub: orders.created
```

---

## Tech Stack

```toml
[dependencies]
axum       = { version = "0.7", features = ["json"] }
sqlx       = { version = "0.8", features = ["runtime-tokio", "postgres", "macros", "uuid"] }
tokio      = { version = "1",   features = ["full"] }
async-nats = "0.35"
serde      = { version = "1",   features = ["derive"] }
serde_json = "1"
tracing    = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
uuid       = { version = "1",   features = ["v4", "serde"] }
reqwest    = { version = "0.12", features = ["json"] }
```

---

## Step 1 — Docker Compose (full stack)

```yaml
# docker-compose.yml
version: "3.9"
services:
  nats:
    image: nats:2.10-alpine
    ports: ["4222:4222"]

  pg-users:
    image: postgres:16-alpine
    environment: { POSTGRES_USER: dev, POSTGRES_PASSWORD: dev, POSTGRES_DB: users }
    healthcheck: { test: ["CMD-SHELL","pg_isready -U dev"], interval: 5s }

  pg-products:
    image: postgres:16-alpine
    environment: { POSTGRES_USER: dev, POSTGRES_PASSWORD: dev, POSTGRES_DB: products }
    healthcheck: { test: ["CMD-SHELL","pg_isready -U dev"], interval: 5s }

  pg-orders:
    image: postgres:16-alpine
    environment: { POSTGRES_USER: dev, POSTGRES_PASSWORD: dev, POSTGRES_DB: orders }
    healthcheck: { test: ["CMD-SHELL","pg_isready -U dev"], interval: 5s }

  users-svc:
    build: { context: ., dockerfile: services/users/Dockerfile }
    ports: ["3001:3001"]
    environment:
      DATABASE_URL: postgresql://dev:dev@pg-users:5432/users
      NATS_URL: nats://nats:4222
      PORT: "3001"
      SERVICE: users
    depends_on: { pg-users: { condition: service_healthy }, nats: { condition: service_started } }

  products-svc:
    build: { context: ., dockerfile: services/products/Dockerfile }
    ports: ["3002:3002"]
    environment:
      DATABASE_URL: postgresql://dev:dev@pg-products:5432/products
      NATS_URL: nats://nats:4222
      PORT: "3002"
      SERVICE: products
    depends_on: { pg-products: { condition: service_healthy }, nats: { condition: service_started } }

  orders-svc:
    build: { context: ., dockerfile: services/orders/Dockerfile }
    ports: ["3003:3003"]
    environment:
      DATABASE_URL: postgresql://dev:dev@pg-orders:5432/orders
      NATS_URL: nats://nats:4222
      PRODUCTS_URL: http://products-svc:3002
      PORT: "3003"
      SERVICE: orders
    depends_on: { pg-orders: { condition: service_healthy }, nats: { condition: service_started } }

  gateway:
    build: { context: ., dockerfile: services/gateway/Dockerfile }
    ports: ["3000:3000"]
    environment:
      USERS_URL: http://users-svc:3001
      PRODUCTS_URL: http://products-svc:3002
      ORDERS_URL: http://orders-svc:3003
```

---

## Step 2 — Shared Tracing Setup

Each service calls this at startup:

```rust
// shared/src/tracing.rs
use tracing_subscriber::{fmt, EnvFilter};

pub fn init(service: &str) {
    let service = service.to_string();
    fmt()
        .json()
        .with_env_filter(EnvFilter::from_default_env()
            .add_directive("info".parse().unwrap()))
        .with_current_span(true)
        .init();
    tracing::info!(service, "Service starting");
}
```

---

## Step 3 — Orders Service (publishes NATS event)

```rust
// services/orders/src/main.rs
use async_nats::Client as NatsClient;
use axum::{extract::State, Json, http::StatusCode, Router, routing::post};
use serde::{Deserialize, Serialize};
use sqlx::PgPool;
use tracing::instrument;
use uuid::Uuid;

#[derive(Deserialize)]
struct CreateOrder { user_id: Uuid, product_id: Uuid, quantity: i32 }

#[derive(Serialize)]
struct OrderCreatedEvent { order_id: Uuid, product_id: Uuid, quantity: i32 }

#[derive(Clone)]
struct AppState { pool: PgPool, nats: NatsClient }

#[instrument(skip(state))]
async fn create_order(
    axum::extract::State(state): axum::extract::State<AppState>,
    Json(body): Json<CreateOrder>,
) -> (StatusCode, Json<serde_json::Value>) {
    let order_id = Uuid::new_v4();

    sqlx::query!(
        "INSERT INTO orders (id, user_id, product_id, quantity) VALUES ($1,$2,$3,$4)",
        order_id, body.user_id, body.product_id, body.quantity
    ).execute(&state.pool).await.unwrap();

    // Publish event to NATS
    let event = OrderCreatedEvent {
        order_id, product_id: body.product_id, quantity: body.quantity,
    };
    state.nats.publish(
        "orders.created",
        serde_json::to_vec(&event).unwrap().into()
    ).await.unwrap();

    tracing::info!(order_id = %order_id, "Order created and event published");
    (StatusCode::CREATED, Json(serde_json::json!({ "order_id": order_id })))
}
```

---

## Step 4 — Products Service (subscribes to NATS)

```rust
// services/products/src/subscriber.rs
use async_nats::Client;
use futures::StreamExt;
use sqlx::PgPool;
use tracing::{info, error};

pub async fn listen_for_orders(nats: Client, pool: PgPool) {
    let mut sub = nats.subscribe("orders.created").await.unwrap();
    info!("Subscribed to orders.created");

    while let Some(msg) = sub.next().await {
        let event: serde_json::Value = serde_json::from_slice(&msg.payload).unwrap_or_default();
        let product_id = event["product_id"].as_str().unwrap_or("");
        let quantity   = event["quantity"].as_i64().unwrap_or(0) as i32;

        match sqlx::query!(
            "UPDATE products SET stock = stock - $1 WHERE id = $2::uuid AND stock >= $1",
            quantity, product_id
        ).execute(&pool).await {
            Ok(r) if r.rows_affected() > 0 =>
                info!(product_id, quantity, "Stock decremented"),
            Ok(_) =>
                error!(product_id, "Insufficient stock — order may be inconsistent"),
            Err(e) =>
                error!(?e, "DB error updating stock"),
        }
    }
}
```

---

## Step 5 — API Gateway

```rust
// services/gateway/src/main.rs
use axum::{Router, routing::{get, any}, extract::{Path, State}, response::IntoResponse};
use reqwest::Client;

#[derive(Clone)]
struct GwState { users: String, products: String, orders: String, client: Client }

async fn proxy(
    State(s): State<GwState>,
    Path((svc, rest)): Path<(String, String)>,
    req: axum::extract::Request,
) -> impl IntoResponse {
    let upstream = match svc.as_str() {
        "users"    => format!("{}/{rest}", s.users),
        "products" => format!("{}/{rest}", s.products),
        "orders"   => format!("{}/{rest}", s.orders),
        _          => return axum::http::StatusCode::NOT_FOUND.into_response(),
    };
    let method = reqwest::Method::from_bytes(req.method().as_str().as_bytes()).unwrap();
    let body   = axum::body::to_bytes(req.into_body(), usize::MAX).await.unwrap();
    match s.client.request(method, &upstream).body(body).send().await {
        Ok(r)  => (r.status(), r.text().await.unwrap_or_default()).into_response(),
        Err(_) => axum::http::StatusCode::BAD_GATEWAY.into_response(),
    }
}
```

---

## Step 6 — Run It

```bash
docker compose up --build
# Gateway at http://localhost:3000
# Create an order:
curl -X POST http://localhost:3000/orders/ \
  -H "Content-Type: application/json" \
  -d '{"user_id":"...","product_id":"...","quantity":2}'
```

---

## Stretch Goals

- [ ] Add OpenTelemetry distributed tracing with a shared `trace_id` across services
- [ ] Add a NATS JetStream stream for durable, at-least-once delivery
- [ ] Add a circuit breaker in the gateway using the `failsafe-rs` crate
- [ ] Write an integration test that spins up all services and verifies end-to-end

---

[Back to All Projects](../index.md){ .md-button }
