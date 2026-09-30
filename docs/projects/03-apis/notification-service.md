# Project: Notification Service

**Track:** 🔌 APIs &nbsp;|&nbsp; **Difficulty:** 🟡 Intermediate &nbsp;|&nbsp; **Crates:** axum, reqwest, serde, tokio

---

## What You'll Build

A webhook-driven notification relay. External services POST events to your endpoint; your service fans them out to all registered targets (Slack webhooks, HTTP URLs, email stubs).

```text
POST /webhooks/receive         {"event":"deploy.success","service":"myapp","env":"prod"}
→ 202 Accepted {"id":"evt-abc123"}

POST /targets/register         {"url":"https://hooks.slack.com/...","name":"prod-alerts"}
→ 201 Created

GET  /events                   → last 50 received events with delivery status
GET  /targets                  → list of registered targets
DELETE /targets/:id            → remove a target
```

---

## Tech Stack

```toml
[dependencies]
axum       = { version = "0.7", features = ["json"] }
tokio      = { version = "1",   features = ["full"] }
serde      = { version = "1",   features = ["derive"] }
serde_json = "1"
reqwest    = { version = "0.12", features = ["json"] }
uuid       = { version = "1",    features = ["v4", "serde"] }
chrono     = { version = "0.4",  features = ["serde"] }
```

---

## Step 1 — Data Model

```rust
// src/model.rs
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct InboundEvent {
    pub id:         Uuid,
    pub payload:    serde_json::Value,
    pub received_at: DateTime<Utc>,
    pub deliveries: Vec<DeliveryRecord>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DeliveryRecord {
    pub target_id:   Uuid,
    pub target_name: String,
    pub status:      DeliveryStatus,
    pub attempted_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum DeliveryStatus {
    Success { http_status: u16 },
    Failed  { reason: String },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Target {
    pub id:   Uuid,
    pub name: String,
    pub url:  String,
}
```

---

## Step 2 — Shared State

```rust
// src/state.rs
use std::collections::{HashMap, VecDeque};
use std::sync::{Arc, Mutex};
use tokio::sync::mpsc;
use crate::model::{InboundEvent, Target};
use uuid::Uuid;

const MAX_EVENTS: usize = 50;

#[derive(Clone)]
pub struct AppState {
    pub events:   Arc<Mutex<VecDeque<InboundEvent>>>,
    pub targets:  Arc<Mutex<HashMap<Uuid, Target>>>,
    pub tx:       mpsc::Sender<InboundEvent>,
}

impl AppState {
    pub fn new(tx: mpsc::Sender<InboundEvent>) -> Self {
        Self {
            events:  Arc::new(Mutex::new(VecDeque::with_capacity(MAX_EVENTS))),
            targets: Arc::new(Mutex::new(HashMap::new())),
            tx,
        }
    }

    pub fn push_event(&self, mut event: InboundEvent) {
        let mut q = self.events.lock().unwrap();
        if q.len() >= MAX_EVENTS { q.pop_front(); }
        q.push_back(event);
    }
}
```

---

## Step 3 — Delivery Worker

```rust
// src/worker.rs
use tokio::sync::mpsc;
use reqwest::Client;
use crate::{model::{InboundEvent, DeliveryRecord, DeliveryStatus}, state::AppState};
use chrono::Utc;

pub async fn run(mut rx: mpsc::Receiver<InboundEvent>, state: AppState) {
    let client = Client::new();
    while let Some(mut event) = rx.recv().await {
        let targets: Vec<_> = state.targets.lock().unwrap().values().cloned().collect();

        for target in targets {
            let status = match client.post(&target.url)
                .json(&event.payload)
                .timeout(std::time::Duration::from_secs(5))
                .send().await
            {
                Ok(r)  => DeliveryStatus::Success { http_status: r.status().as_u16() },
                Err(e) => DeliveryStatus::Failed  { reason: e.to_string() },
            };
            event.deliveries.push(DeliveryRecord {
                target_id:    target.id,
                target_name:  target.name,
                status,
                attempted_at: Utc::now(),
            });
        }
        state.push_event(event);
    }
}
```

---

## Step 4 — Handlers & Main

```rust
// src/handlers.rs
use axum::{extract::State, http::StatusCode, Json};
use serde::Deserialize;
use uuid::Uuid;
use chrono::Utc;
use crate::{model::{InboundEvent, Target}, state::AppState};

#[derive(Deserialize)]
pub struct RegisterTarget { pub name: String, pub url: String }

pub async fn receive(State(s): State<AppState>, Json(payload): Json<serde_json::Value>)
    -> (StatusCode, Json<serde_json::Value>)
{
    let event = InboundEvent {
        id: Uuid::new_v4(), payload,
        received_at: Utc::now(), deliveries: vec![],
    };
    let id = event.id;
    let _ = s.tx.send(event).await;
    (StatusCode::ACCEPTED, Json(serde_json::json!({ "id": id })))
}

pub async fn register(State(s): State<AppState>, Json(b): Json<RegisterTarget>)
    -> (StatusCode, Json<Target>)
{
    let t = Target { id: Uuid::new_v4(), name: b.name, url: b.url };
    s.targets.lock().unwrap().insert(t.id, t.clone());
    (StatusCode::CREATED, Json(t))
}

pub async fn list_events(State(s): State<AppState>) -> Json<Vec<InboundEvent>> {
    Json(s.events.lock().unwrap().iter().cloned().collect())
}

pub async fn list_targets(State(s): State<AppState>) -> Json<Vec<Target>> {
    Json(s.targets.lock().unwrap().values().cloned().collect())
}
```

```rust
// src/main.rs
mod model;
mod state;
mod worker;
mod handlers;

use axum::{Router, routing::{get, post}};
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, rx) = mpsc::channel(256);
    let state = state::AppState::new(tx);

    tokio::spawn(worker::run(rx, state.clone()));

    let app = Router::new()
        .route("/webhooks/receive",  post(handlers::receive))
        .route("/targets/register",  post(handlers::register))
        .route("/targets",           get(handlers::list_targets))
        .route("/events",            get(handlers::list_events))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("Notification service at http://localhost:3000");
    axum::serve(listener, app).await.unwrap();
}
```

---

## Step 5 — Test It

```bash
cargo run

# Register a target (use https://webhook.site for a test URL)
curl -X POST http://localhost:3000/targets/register \
  -H "Content-Type: application/json" \
  -d '{"name":"test","url":"https://webhook.site/your-uuid"}'

# Send an event
curl -X POST http://localhost:3000/webhooks/receive \
  -H "Content-Type: application/json" \
  -d '{"event":"deploy.success","service":"myapp"}'

# Check delivery status
curl http://localhost:3000/events
```

---

## Stretch Goals

- [ ] Add retry logic: retry failed deliveries up to 3 times with exponential backoff
- [ ] Add event filtering: targets can subscribe to specific event types
- [ ] Add HMAC signature verification for incoming webhooks
- [ ] Persist events and targets to SQLite using `sqlx`

---

[Back to All Projects](../index.md){ .md-button }


[Next Track: Cloud Projects](../04-cloud/containerised-microservice.md){ .md-button .md-button--primary }
