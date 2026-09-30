# Project: Event-driven Analytics

**Track:** 🏢 Enterprise Apps &nbsp;|&nbsp; **Difficulty:** 🔴 Advanced &nbsp;|&nbsp; **Crates:** axum, tokio, metrics, serde

---

## What You'll Build

A high-throughput event ingestion service that accepts events at `POST /events`, processes them asynchronously through a `tokio::sync::mpsc` channel, aggregates page views and user activity in real time, and exposes the results plus Prometheus metrics.

Target: **10,000 events/second** on a single process.

```text
POST /events           {"type":"page_view","url":"/home","user_id":"u1"}
→ 202 Accepted

GET  /analytics/pages  → [{"url":"/home","views":1423}, ...]
GET  /analytics/users  → [{"hour":"2024-06-12T14:00","unique_users":38}, ...]
GET  /metrics          → Prometheus format
```

---

## Tech Stack

```toml
[dependencies]
axum                       = { version = "0.7", features = ["json"] }
tokio                      = { version = "1",   features = ["full"] }
serde                      = { version = "1",   features = ["derive"] }
serde_json                 = "1"
metrics                    = "0.23"
metrics-exporter-prometheus = "0.15"
chrono                     = { version = "0.4", features = ["serde"] }
dashmap                    = "5"
```

---

## Step 1 — Event Model

```rust
// src/model.rs
use serde::{Deserialize, Serialize};
use chrono::{DateTime, Utc};

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct Event {
    #[serde(rename = "type")]
    pub kind:       String,
    pub url:        Option<String>,
    pub user_id:    Option<String>,
    #[serde(default = "Utc::now")]
    pub timestamp:  DateTime<Utc>,
}

#[derive(Serialize)]
pub struct PageStat  { pub url: String, pub views: u64 }

#[derive(Serialize)]
pub struct HourStat  { pub hour: String, pub unique_users: usize }
```

---

## Step 2 — Aggregator (background task)

```rust
// src/aggregator.rs
use std::collections::{HashMap, HashSet};
use dashmap::DashMap;
use std::sync::Arc;
use tokio::sync::mpsc;
use crate::model::Event;

#[derive(Clone)]
pub struct Stats {
    pub page_views:   Arc<DashMap<String, u64>>,
    pub hourly_users: Arc<DashMap<String, HashSet<String>>>,
}

impl Stats {
    pub fn new() -> Self {
        Self {
            page_views:   Arc::new(DashMap::new()),
            hourly_users: Arc::new(DashMap::new()),
        }
    }
}

pub async fn run(mut rx: mpsc::Receiver<Event>, stats: Stats) {
    while let Some(event) = rx.recv().await {
        metrics::counter!("events_processed_total").increment(1);

        if event.kind == "page_view" {
            if let Some(url) = &event.url {
                *stats.page_views.entry(url.clone()).or_insert(0) += 1;
            }
        }

        if let Some(user_id) = &event.user_id {
            let hour = event.timestamp.format("%Y-%m-%dT%H:00").to_string();
            stats.hourly_users
                .entry(hour)
                .or_insert_with(HashSet::new)
                .insert(user_id.clone());
        }
    }
}
```

---

## Step 3 — Handlers

```rust
// src/handlers.rs
use axum::{extract::State, http::StatusCode, Json};
use tokio::sync::mpsc;
use crate::{model::{Event, PageStat, HourStat}, aggregator::Stats};

#[derive(Clone)]
pub struct AppState {
    pub tx:    mpsc::Sender<Event>,
    pub stats: Stats,
}

pub async fn ingest(
    State(s): State<AppState>,
    Json(event): Json<Event>,
) -> StatusCode {
    metrics::counter!("events_received_total").increment(1);
    let _ = s.tx.try_send(event);  // non-blocking: drop if queue full
    StatusCode::ACCEPTED
}

pub async fn page_stats(State(s): State<AppState>) -> Json<Vec<PageStat>> {
    let mut pages: Vec<PageStat> = s.stats.page_views.iter()
        .map(|e| PageStat { url: e.key().clone(), views: *e.value() })
        .collect();
    pages.sort_by(|a, b| b.views.cmp(&a.views));
    Json(pages.into_iter().take(20).collect())
}

pub async fn hourly_stats(State(s): State<AppState>) -> Json<Vec<HourStat>> {
    let mut hours: Vec<HourStat> = s.stats.hourly_users.iter()
        .map(|e| HourStat { hour: e.key().clone(), unique_users: e.value().len() })
        .collect();
    hours.sort_by(|a, b| b.hour.cmp(&a.hour));
    Json(hours.into_iter().take(24).collect())
}
```

---

## Step 4 — Prometheus Metrics + Main

```rust
// src/main.rs
mod model;
mod aggregator;
mod handlers;

use axum::{Router, routing::{get, post}};
use metrics_exporter_prometheus::PrometheusBuilder;
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    // Set up Prometheus metrics exporter
    let prometheus_handle = PrometheusBuilder::new()
        .install_recorder().unwrap();

    let (tx, rx) = mpsc::channel(10_000);
    let stats = aggregator::Stats::new();

    tokio::spawn(aggregator::run(rx, stats.clone()));

    let state = handlers::AppState { tx, stats };

    let app = Router::new()
        .route("/events",           post(handlers::ingest))
        .route("/analytics/pages",  get(handlers::page_stats))
        .route("/analytics/users",  get(handlers::hourly_stats))
        .route("/metrics", get(move || {
            let handle = prometheus_handle.clone();
            async move { handle.render() }
        }))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("Analytics service at http://localhost:3000");
    axum::serve(listener, app).await.unwrap();
}
```

---

## Step 5 — Load Test

```bash
cargo run

# Install oha (HTTP load tester written in Rust)
cargo install oha

# Fire 50,000 requests at 1000 RPS
oha -n 50000 -c 100 -m POST \
  -H "Content-Type: application/json" \
  -d '{"type":"page_view","url":"/home","user_id":"u1"}' \
  http://localhost:3000/events

# Check results
curl http://localhost:3000/analytics/pages
curl http://localhost:3000/metrics
```

---

## Stretch Goals

- [ ] Persist aggregated stats to PostgreSQL every 60 seconds
- [ ] Add a `GET /analytics/realtime` WebSocket endpoint streaming live counters
- [ ] Add event validation: reject unknown event types with 400
- [ ] Add a Grafana dashboard config that visualises the Prometheus metrics

---

[Back to All Projects](../index.md){ .md-button }


[Next Project: Microservice Mesh](microservice-mesh.md){ .md-button .md-button--primary }
