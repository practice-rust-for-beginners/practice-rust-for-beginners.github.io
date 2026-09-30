# Project: Link Shortener

**Track:** 🌐 Web Development &nbsp;|&nbsp; **Difficulty:** 🟡 Intermediate &nbsp;|&nbsp; **Crates:** axum, tokio, serde, rand

---

## What You'll Build

A URL shortener service with redirect tracking, click analytics, and an expiry system. Every short URL tracks how many times it's been clicked.

```text
POST /shorten              {"url":"https://rust-lang.org","ttl_hours":24}
→ {"short":"http://localhost:3000/r/aB3xY","code":"aB3xY"}

GET  /r/aB3xY              → 302 redirect to https://rust-lang.org
GET  /stats/aB3xY          → {"url":"https://rust-lang.org","clicks":17,"created_at":"...","expires_at":"..."}
GET  /stats                → [{"code":"aB3xY","clicks":17}, ...]
DELETE /r/aB3xY            → 204 No Content
```

---

## Tech Stack

| Component | Crate |
|-----------|-------|
| HTTP server | `axum 0.7` |
| Short code generation | `rand 0.8` |
| Shared state | `Arc<RwLock<HashMap>>` |
| Serialisation | `serde` |
| Timestamps | `chrono` |

```toml
[dependencies]
axum    = { version = "0.7", features = ["json"] }
tokio   = { version = "1",   features = ["full"] }
serde   = { version = "1",   features = ["derive"] }
serde_json = "1"
rand    = "0.8"
chrono  = { version = "0.4", features = ["serde"] }
```

---

## Step 1 — Data Model

```rust
// src/store.rs
use chrono::{DateTime, Utc, Duration};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::{Arc, RwLock};

#[derive(Debug, Clone, Serialize)]
pub struct ShortLink {
    pub code:       String,
    pub url:        String,
    pub clicks:     u64,
    pub created_at: DateTime<Utc>,
    pub expires_at: Option<DateTime<Utc>>,
}

pub type Store = Arc<RwLock<HashMap<String, ShortLink>>>;

pub fn new_store() -> Store {
    Arc::new(RwLock::new(HashMap::new()))
}

pub fn generate_code() -> String {
    use rand::Rng;
    let mut rng = rand::thread_rng();
    (0..6).map(|_| {
        let idx = rng.gen_range(0..62);
        "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
            .chars().nth(idx).unwrap()
    }).collect()
}
```

---

## Step 2 — Handlers

```rust
// src/handlers.rs
use axum::{
    extract::{Path, State},
    http::{StatusCode, HeaderMap, header},
    Json, response::IntoResponse,
};
use serde::{Deserialize, Serialize};
use chrono::{Utc, Duration};
use crate::store::{Store, ShortLink, generate_code};

#[derive(Deserialize)]
pub struct ShortenRequest {
    pub url:       String,
    pub ttl_hours: Option<i64>,
}

#[derive(Serialize)]
pub struct ShortenResponse {
    pub short: String,
    pub code:  String,
}

pub async fn shorten(
    State(store): State<Store>,
    Json(req): Json<ShortenRequest>,
) -> impl IntoResponse {
    let code = generate_code();
    let expires_at = req.ttl_hours.map(|h| Utc::now() + Duration::hours(h));

    let link = ShortLink {
        code:       code.clone(),
        url:        req.url,
        clicks:     0,
        created_at: Utc::now(),
        expires_at,
    };

    store.write().unwrap().insert(code.clone(), link);

    (StatusCode::CREATED, Json(ShortenResponse {
        short: format!("http://localhost:3000/r/{code}"),
        code,
    }))
}

pub async fn redirect(
    State(store): State<Store>,
    Path(code): Path<String>,
) -> impl IntoResponse {
    let mut map = store.write().unwrap();
    match map.get_mut(&code) {
        None => StatusCode::NOT_FOUND.into_response(),
        Some(link) => {
            // Check expiry
            if let Some(exp) = link.expires_at {
                if Utc::now() > exp {
                    map.remove(&code);
                    return StatusCode::GONE.into_response();
                }
            }
            link.clicks += 1;
            let url = link.url.clone();
            let mut headers = HeaderMap::new();
            headers.insert(header::LOCATION, url.parse().unwrap());
            (StatusCode::FOUND, headers).into_response()
        }
    }
}

pub async fn stats_one(
    State(store): State<Store>,
    Path(code): Path<String>,
) -> impl IntoResponse {
    match store.read().unwrap().get(&code).cloned() {
        Some(link) => Json(link).into_response(),
        None       => StatusCode::NOT_FOUND.into_response(),
    }
}

pub async fn stats_all(State(store): State<Store>) -> Json<Vec<ShortLink>> {
    let mut links: Vec<ShortLink> = store.read().unwrap().values().cloned().collect();
    links.sort_by(|a, b| b.clicks.cmp(&a.clicks));
    Json(links)
}

pub async fn delete(
    State(store): State<Store>,
    Path(code): Path<String>,
) -> StatusCode {
    if store.write().unwrap().remove(&code).is_some() {
        StatusCode::NO_CONTENT
    } else {
        StatusCode::NOT_FOUND
    }
}
```

---

## Step 3 — Router & Main

```rust
// src/main.rs
mod store;
mod handlers;

use axum::{Router, routing::{get, post, delete}};

#[tokio::main]
async fn main() {
    let store = store::new_store();

    let app = Router::new()
        .route("/shorten",     post(handlers::shorten))
        .route("/r/:code",     get(handlers::redirect).delete(handlers::delete))
        .route("/stats",       get(handlers::stats_all))
        .route("/stats/:code", get(handlers::stats_one))
        .with_state(store);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("Link shortener running at http://localhost:3000");
    axum::serve(listener, app).await.unwrap();
}
```

---

## Step 4 — Test It

```bash
cargo run

# Shorten a URL
curl -X POST http://localhost:3000/shorten \
  -H "Content-Type: application/json" \
  -d '{"url":"https://www.rust-lang.org","ttl_hours":24}'

# Follow the redirect
curl -L http://localhost:3000/r/aB3xY

# Check stats
curl http://localhost:3000/stats/aB3xY
```

---

## Stretch Goals

- [ ] Persist the store to a JSON file on shutdown and reload on startup
- [ ] Add a simple HTML front-end form to shorten URLs in the browser
- [ ] Add a background task that sweeps expired links every 5 minutes
- [ ] Add custom slug support: `POST /shorten {"url":"...","custom":"my-link"}`

---

[Back to All Projects](../index.md){ .md-button }


[Next Project: Real-time Chat](realtime-chat.md){ .md-button .md-button--primary }
