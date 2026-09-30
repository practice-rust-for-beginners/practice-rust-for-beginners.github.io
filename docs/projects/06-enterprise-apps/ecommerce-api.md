# Project: E-commerce API

**Track:** 🏢 Enterprise Apps &nbsp;|&nbsp; **Difficulty:** 🔴 Advanced &nbsp;|&nbsp; **Crates:** axum, sqlx, tokio, jsonwebtoken, argon2

---

## What You'll Build

A production-ready e-commerce backend: user auth with JWT, product catalogue, shopping cart, and transactional order processing — all backed by PostgreSQL.

```text
POST /auth/register       {"email":"alice@example.com","password":"secret"}
POST /auth/login          → {"access_token":"eyJ...","refresh_token":"..."}

GET  /products            → paginated product list
POST /products            → create (admin only)

POST /cart/add            {"product_id":1,"quantity":2}
GET  /cart                → current cart total
DELETE /cart/:product_id  → remove item

POST /orders              → checkout (deducts stock in TX)
GET  /orders              → my order history
```

---

## Tech Stack

```toml
[dependencies]
axum         = { version = "0.7", features = ["json"] }
sqlx         = { version = "0.8", features = ["runtime-tokio", "postgres", "macros", "uuid", "chrono"] }
tokio        = { version = "1",   features = ["full"] }
serde        = { version = "1",   features = ["derive"] }
serde_json   = "1"
jsonwebtoken = "9"
argon2       = "0.5"
uuid         = { version = "1",   features = ["v4", "serde"] }
chrono       = { version = "0.4", features = ["serde"] }
dotenvy      = "0.15"
tower-http   = { version = "0.5", features = ["cors", "trace"] }
```

---

## Step 1 — Database Migrations

```sql
-- migrations/001_users.sql
CREATE TABLE users (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email         VARCHAR(255) UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    role          VARCHAR(20) NOT NULL DEFAULT 'user',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- migrations/002_products.sql
CREATE TABLE products (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255) NOT NULL,
    description TEXT,
    price       NUMERIC(10,2) NOT NULL,
    stock       INT NOT NULL DEFAULT 0
);

-- migrations/003_cart.sql
CREATE TABLE cart_items (
    user_id    UUID REFERENCES users(id) ON DELETE CASCADE,
    product_id UUID REFERENCES products(id) ON DELETE CASCADE,
    quantity   INT NOT NULL,
    PRIMARY KEY (user_id, product_id)
);

-- migrations/004_orders.sql
CREATE TABLE orders (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id    UUID REFERENCES users(id),
    total      NUMERIC(10,2) NOT NULL,
    status     VARCHAR(20) NOT NULL DEFAULT 'confirmed',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE order_items (
    order_id   UUID REFERENCES orders(id),
    product_id UUID REFERENCES products(id),
    quantity   INT NOT NULL,
    unit_price NUMERIC(10,2) NOT NULL,
    PRIMARY KEY (order_id, product_id)
);
```

---

## Step 2 — Auth Middleware

```rust
// src/auth.rs
use axum::{extract::FromRequestParts, http::{request::Parts, StatusCode}};
use jsonwebtoken::{decode, DecodingKey, Validation, Algorithm};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct Claims {
    pub sub:  Uuid,
    pub role: String,
    pub exp:  usize,
}

#[axum::async_trait]
impl<S: Send + Sync> FromRequestParts<S> for Claims {
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(parts: &mut Parts, _: &S) -> Result<Self, Self::Rejection> {
        let auth = parts.headers.get("Authorization")
            .and_then(|v| v.to_str().ok())
            .and_then(|v| v.strip_prefix("Bearer "))
            .ok_or((StatusCode::UNAUTHORIZED, "Missing token"))?;

        let secret = std::env::var("JWT_SECRET").unwrap_or("dev-secret".into());
        decode::<Claims>(auth,
            &DecodingKey::from_secret(secret.as_bytes()),
            &Validation::new(Algorithm::HS256))
        .map(|d| d.claims)
        .map_err(|_| (StatusCode::UNAUTHORIZED, "Invalid token"))
    }
}

pub fn create_token(user_id: Uuid, role: &str, secret: &str) -> String {
    use jsonwebtoken::{encode, EncodingKey, Header};
    let exp = (chrono::Utc::now() + chrono::Duration::hours(24)).timestamp() as usize;
    encode(&Header::default(),
        &Claims { sub: user_id, role: role.to_string(), exp },
        &EncodingKey::from_secret(secret.as_bytes())
    ).unwrap()
}
```

---

## Step 3 — Order Checkout with Transaction

```rust
// src/handlers/orders.rs
use axum::{extract::State, http::StatusCode, Json};
use sqlx::PgPool;
use uuid::Uuid;
use crate::auth::Claims;

pub async fn checkout(
    claims: Claims,
    State(pool): State<PgPool>,
) -> Result<Json<serde_json::Value>, (StatusCode, String)> {
    let user_id = claims.sub;
    let mut tx = pool.begin().await.map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

    // Fetch cart items with product prices
    let items = sqlx::query!(
        "SELECT ci.product_id, ci.quantity, p.price, p.stock, p.name
         FROM cart_items ci JOIN products p ON p.id = ci.product_id
         WHERE ci.user_id = $1", user_id
    ).fetch_all(&mut *tx).await.map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

    if items.is_empty() {
        return Err((StatusCode::BAD_REQUEST, "Cart is empty".into()));
    }

    // Verify stock and compute total
    for item in &items {
        if item.stock < item.quantity {
            return Err((StatusCode::CONFLICT, format!("Insufficient stock for '{}'", item.name)));
        }
    }
    let total: f64 = items.iter().map(|i| i.price.to_string().parse::<f64>().unwrap_or(0.0) * i.quantity as f64).sum();

    // Create order
    let order_id = Uuid::new_v4();
    sqlx::query!("INSERT INTO orders (id, user_id, total) VALUES ($1, $2, $3)",
        order_id, user_id, total as f64
    ).execute(&mut *tx).await.map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

    // Insert order items + deduct stock
    for item in &items {
        sqlx::query!("INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES ($1,$2,$3,$4)",
            order_id, item.product_id, item.quantity, item.price
        ).execute(&mut *tx).await.map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

        sqlx::query!("UPDATE products SET stock = stock - $1 WHERE id = $2",
            item.quantity, item.product_id
        ).execute(&mut *tx).await.map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;
    }

    // Clear cart
    sqlx::query!("DELETE FROM cart_items WHERE user_id = $1", user_id)
        .execute(&mut *tx).await.map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

    tx.commit().await.map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

    Ok(Json(serde_json::json!({ "order_id": order_id, "total": total, "status": "confirmed" })))
}
```

---

## Step 4 — Router Setup

```rust
// src/main.rs (excerpt)
let app = Router::new()
    .route("/auth/register", post(handlers::auth::register))
    .route("/auth/login",    post(handlers::auth::login))
    .route("/products",      get(handlers::products::list).post(handlers::products::create))
    .route("/products/:id",  get(handlers::products::get_one))
    .route("/cart",          get(handlers::cart::get_cart))
    .route("/cart/add",      post(handlers::cart::add_item))
    .route("/cart/:id",      delete(handlers::cart::remove_item))
    .route("/orders",        get(handlers::orders::list).post(handlers::orders::checkout))
    .layer(TraceLayer::new_for_http())
    .with_state(pool);
```

---

## Stretch Goals

- [ ] Add refresh token rotation
- [ ] Add email verification on registration (send via SMTP or `lettre` crate)
- [ ] Add `GET /admin/orders` for admins to see all orders
- [ ] Add product images stored in S3 via `aws-sdk-s3`

---

[Back to All Projects](../index.md){ .md-button }


[Next Project: Event-driven Analytics](event-driven-analytics.md){ .md-button .md-button--primary }
