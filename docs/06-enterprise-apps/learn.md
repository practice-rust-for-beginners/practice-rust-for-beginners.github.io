# Track 6 — Enterprise Apps: Learn

Track 6 is the capstone track. You'll combine everything from the previous tracks and add the patterns used in large-scale production systems: persistent databases, distributed messaging, structured observability, authentication, and WebAssembly compilation.

**Prerequisites:** All previous tracks.

---

## 1. Databases with SQLx

**SQLx** is a compile-time checked, async SQL library for PostgreSQL, MySQL, and SQLite.

```toml
[dependencies]
sqlx  = { version = "0.8", features = ["runtime-tokio", "postgres", "macros", "chrono", "uuid"] }
tokio = { version = "1",   features = ["full"] }
```

### Connecting & querying

```rust
use sqlx::PgPool;

#[tokio::main]
async fn main() -> Result<(), sqlx::Error> {
    let pool = PgPool::connect(&std::env::var("DATABASE_URL").unwrap()).await?;

    // Compile-time checked query
    let rows = sqlx::query!("SELECT id, name, email FROM users WHERE active = true")
        .fetch_all(&pool).await?;

    for row in rows {
        println!("{}: {}", row.id, row.name);
    }
    Ok(())
}
```

### Typed queries with structs

```rust
#[derive(sqlx::FromRow)]
struct User { id: i32, name: String, email: String }

let users: Vec<User> = sqlx::query_as!(User,
    "SELECT id, name, email FROM users WHERE id = $1", user_id
).fetch_all(&pool).await?;
```

### Transactions

```rust
let mut tx = pool.begin().await?;

sqlx::query!("INSERT INTO orders (user_id, total) VALUES ($1, $2)", user_id, total)
    .execute(&mut *tx).await?;

sqlx::query!("UPDATE inventory SET quantity = quantity - $1 WHERE product_id = $2",
             quantity, product_id)
    .execute(&mut *tx).await?;

tx.commit().await?;
```

---

## 2. Database Migrations

Use **SQLx CLI** to manage schema migrations:

```bash
cargo install sqlx-cli
sqlx migrate add create_users_table
# creates migrations/20240101000000_create_users_table.sql
```

```sql
-- migrations/20240101000000_create_users_table.sql
CREATE TABLE users (
    id         SERIAL PRIMARY KEY,
    name       VARCHAR(255) NOT NULL,
    email      VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

```bash
sqlx migrate run          # apply all pending migrations
sqlx migrate revert       # roll back the last migration
```

Apply migrations at application startup:

```rust
sqlx::migrate!("./migrations").run(&pool).await?;
```

---

## 3. Diesel ORM

**Diesel** is a compile-time-safe ORM. Use it when you want a more ActiveRecord-style experience.

```toml
[dependencies]
diesel = { version = "2", features = ["postgres", "r2d2"] }
```

```rust
use diesel::prelude::*;

#[derive(Queryable, Selectable)]
#[diesel(table_name = users)]
struct User { id: i32, name: String }

#[derive(Insertable)]
#[diesel(table_name = users)]
struct NewUser<'a> { name: &'a str, email: &'a str }

fn create_user(conn: &mut PgConnection, name: &str, email: &str) -> User {
    diesel::insert_into(users::table)
        .values(NewUser { name, email })
        .returning(User::as_returning())
        .get_result(conn)
        .expect("Error creating user")
}
```

---

## 4. Structured Logging with `tracing`

`tracing` is the standard instrumentation library for Rust services.

```toml
[dependencies]
tracing            = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
```

```rust
use tracing::{info, warn, error, instrument};

#[instrument(skip(pool), fields(user_id = %id))]
async fn get_user(id: i32, pool: &PgPool) -> Result<User, sqlx::Error> {
    info!("Fetching user from database");
    let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", id)
        .fetch_one(pool).await?;
    info!(name = %user.name, "User found");
    Ok(user)
}

fn init_tracing() {
    tracing_subscriber::fmt()
        .json()                     // structured JSON logs
        .with_env_filter("info")    // RUST_LOG=debug overrides
        .init();
}
```

---

## 5. OpenTelemetry — Distributed Tracing

```toml
[dependencies]
opentelemetry      = "0.23"
opentelemetry-otlp = { version = "0.16", features = ["tokio"] }
tracing-opentelemetry = "0.24"
```

```rust
use opentelemetry::global;
use opentelemetry_otlp::WithExportConfig;

fn init_otel() {
    let tracer = opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(
            opentelemetry_otlp::new_exporter()
                .tonic()
                .with_endpoint("http://localhost:4317"),
        )
        .install_batch(opentelemetry_sdk::runtime::Tokio)
        .expect("OTLP tracer setup failed");

    let otel_layer = tracing_opentelemetry::layer().with_tracer(tracer);
    // add to tracing_subscriber::registry()
}
```

---

## 6. Message Queues — Kafka with `rdkafka`

```toml
[dependencies]
rdkafka = { version = "0.36", features = ["tokio"] }
```

```rust
use rdkafka::producer::{FutureProducer, FutureRecord};
use rdkafka::ClientConfig;

// Producer
let producer: FutureProducer = ClientConfig::new()
    .set("bootstrap.servers", "localhost:9092")
    .create()?;

producer.send(
    FutureRecord::to("orders")
        .key("order-123")
        .payload(r#"{"order_id":123,"total":99.99}"#),
    std::time::Duration::from_secs(5),
).await?;

// Consumer
use rdkafka::consumer::{StreamConsumer, Consumer};

let consumer: StreamConsumer = ClientConfig::new()
    .set("group.id",           "order-processor")
    .set("bootstrap.servers",  "localhost:9092")
    .create()?;

consumer.subscribe(&["orders"])?;
let mut stream = consumer.stream();
while let Some(msg) = stream.next().await {
    let m = msg?;
    println!("{:?}", m.payload_view::<str>());
}
```

---

## 7. NATS Messaging

NATS is simpler than Kafka — great for pub/sub and request/reply.

```toml
[dependencies]
async-nats = "0.35"
```

```rust
use async_nats::connect;

// Publish
let client = connect("nats://localhost:4222").await?;
client.publish("events.user.created", b"...".into()).await?;

// Subscribe
let mut subscriber = client.subscribe("events.>").await?;
while let Some(msg) = subscriber.next().await {
    println!("Subject: {}, Data: {:?}", msg.subject, msg.payload);
}
```

---

## 8. JWT & OAuth2 Authentication

```toml
[dependencies]
jsonwebtoken = "9"
oauth2       = "4"
```

```rust
use jsonwebtoken::{encode, decode, Header, Validation, EncodingKey, DecodingKey, Algorithm};

#[derive(serde::Serialize, serde::Deserialize)]
struct Claims {
    sub:   String,
    roles: Vec<String>,
    exp:   usize,
    iat:   usize,
}

fn sign_jwt(user_id: &str, roles: Vec<String>, secret: &[u8]) -> String {
    let now = chrono::Utc::now().timestamp() as usize;
    encode(
        &Header::default(),
        &Claims { sub: user_id.to_owned(), roles, exp: now + 3600, iat: now },
        &EncodingKey::from_secret(secret),
    ).unwrap()
}
```

---

## 9. WebAssembly (WASM)

Compile Rust to WASM for the browser or WASI environments:

```bash
rustup target add wasm32-unknown-unknown
cargo install wasm-pack

# Browser WASM
wasm-pack build --target web

# WASI (serverside WASM)
rustup target add wasm32-wasi
cargo build --target wasm32-wasi
```

```rust
// src/lib.rs
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn fibonacci(n: u32) -> u32 {
    match n {
        0 => 0, 1 => 1,
        _ => fibonacci(n - 1) + fibonacci(n - 2),
    }
}
```

---

## 10. Performance Profiling

```toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }
```

```rust
// benches/my_bench.rs
use criterion::{criterion_group, criterion_main, Criterion, BenchmarkId};

fn bench_fibonacci(c: &mut Criterion) {
    for n in [10u32, 20, 30].iter() {
        c.bench_with_input(
            BenchmarkId::new("fibonacci", n),
            n,
            |b, &n| b.iter(|| fibonacci(n)),
        );
    }
}

criterion_group!(benches, bench_fibonacci);
criterion_main!(benches);
```

```bash
cargo bench --bench my_bench
```

---

## What's Next?

You've completed all 6 tracks. You now have everything you need to build production Rust applications at scale.

[Go to Practice :fontawesome-solid-dumbbell:](practice.md){ .md-button .md-button--primary }
[Jump to Challenges :fontawesome-solid-trophy:](challenges.md){ .md-button }
