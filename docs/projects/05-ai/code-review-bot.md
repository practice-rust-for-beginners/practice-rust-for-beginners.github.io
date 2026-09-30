# Project: Code Review Bot

**Track:** 🤖 AI &nbsp;|&nbsp; **Difficulty:** 🔴 Advanced &nbsp;|&nbsp; **Crates:** axum, async-openai, tokio, serde

---

## What You'll Build

An Axum REST API that takes Rust source code, runs `cargo clippy` on it in a sandboxed temporary directory, and sends it to an LLM for a structured code review. Returns a JSON response with issues, suggestions, and a score.

```text
POST /review
{
  "code": "fn main() { let x = 5; println!(\"{}\", x); }",
  "level": "beginner"
}

→ {
    "clippy": ["warning: unused variable `x`"],
    "issues": [{"severity":"warning","message":"Consider using `let _x` to suppress warning"}],
    "suggestions": ["Use string interpolation: println!(\"{x}\")"],
    "score": 7.5,
    "summary": "Clean code for a beginner. Minor style improvements available."
  }
```

---

## Tech Stack

```toml
[dependencies]
axum         = { version = "0.7", features = ["json"] }
tokio        = { version = "1",   features = ["full"] }
serde        = { version = "1",   features = ["derive"] }
serde_json   = "1"
async-openai = "0.23"
sha2         = "0.10"
hex          = "0.4"
```

---

## Step 1 — Request & Response Types

```rust
// src/model.rs
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
pub struct ReviewRequest {
    pub code:  String,
    #[serde(default = "default_level")]
    pub level: String,
}
fn default_level() -> String { "intermediate".into() }

#[derive(Serialize)]
pub struct ReviewResponse {
    pub clippy:      Vec<String>,
    pub issues:      Vec<Issue>,
    pub suggestions: Vec<String>,
    pub score:       f32,
    pub summary:     String,
}

#[derive(Serialize, Deserialize)]
pub struct Issue {
    pub severity: String,
    pub message:  String,
}
```

---

## Step 2 — Clippy Runner

```rust
// src/clippy.rs
use std::{fs, process::Command};
use tempfile::TempDir;

pub fn run_clippy(code: &str) -> Vec<String> {
    let dir = TempDir::new().unwrap();
    let src  = dir.path().join("src");
    fs::create_dir_all(&src).unwrap();
    fs::write(src.join("main.rs"), code).unwrap();
    fs::write(dir.path().join("Cargo.toml"), r#"
[package]
name = "review_target"
version = "0.1.0"
edition = "2021"
"#).unwrap();

    let output = Command::new("cargo")
        .args(["clippy", "--message-format=short", "--", "-D", "warnings"])
        .current_dir(dir.path())
        .output()
        .unwrap_or_default();

    let stderr = String::from_utf8_lossy(&output.stderr);
    stderr.lines()
        .filter(|l| l.contains("warning") || l.contains("error"))
        .map(String::from)
        .collect()
}
```

Add `tempfile = "3"` to `Cargo.toml` dependencies.

---

## Step 3 — LLM Review

```rust
// src/ai_review.rs
use async_openai::{Client, types::{CreateChatCompletionRequestArgs, ChatCompletionRequestUserMessageArgs}};
use crate::model::{Issue, ReviewResponse};

const SYSTEM: &str = r#"You are an expert Rust code reviewer.
Analyse the provided Rust code and respond with ONLY valid JSON in this exact schema:
{
  "issues": [{"severity": "error|warning|info", "message": "..."}],
  "suggestions": ["..."],
  "score": 0.0,
  "summary": "..."
}
Score 0-10. Be concise. Focus on Rust idioms, ownership, error handling."#;

pub async fn review(code: &str, level: &str, clippy: &[String]) -> ReviewResponse {
    let clippy_section = if clippy.is_empty() {
        "Clippy: no warnings".to_string()
    } else {
        format!("Clippy warnings:\n{}", clippy.join("\n"))
    };

    let prompt = format!(
        "Reviewer level: {level}\n{clippy_section}\n\nCode to review:\n```rust\n{code}\n```"
    );

    let client = Client::new();
    let request = CreateChatCompletionRequestArgs::default()
        .model("gpt-4o-mini")
        .messages([
            async_openai::types::ChatCompletionRequestSystemMessageArgs::default()
                .content(SYSTEM).build().unwrap().into(),
            ChatCompletionRequestUserMessageArgs::default()
                .content(prompt).build().unwrap().into(),
        ])
        .response_format(async_openai::types::ChatCompletionResponseFormat {
            r#type: async_openai::types::ChatCompletionResponseFormatType::JsonObject,
        })
        .build().unwrap();

    let resp = client.chat().create(request).await.unwrap();
    let json_text = resp.choices[0].message.content.as_deref().unwrap_or("{}");
    let parsed: serde_json::Value = serde_json::from_str(json_text).unwrap_or_default();

    ReviewResponse {
        clippy:      clippy.to_vec(),
        issues:      serde_json::from_value(parsed["issues"].clone()).unwrap_or_default(),
        suggestions: serde_json::from_value(parsed["suggestions"].clone()).unwrap_or_default(),
        score:       parsed["score"].as_f64().unwrap_or(5.0) as f32,
        summary:     parsed["summary"].as_str().unwrap_or("").to_string(),
    }
}
```

---

## Step 4 — Cache + Handler

```rust
// src/handler.rs
use axum::{extract::State, http::StatusCode, Json};
use std::{collections::HashMap, sync::{Arc, RwLock}, time::{Instant, Duration}};
use sha2::{Sha256, Digest};
use crate::{model::{ReviewRequest, ReviewResponse}, clippy::run_clippy, ai_review::review};

#[derive(Clone)]
struct CacheEntry { response: ReviewResponse, created: Instant }
pub type Cache = Arc<RwLock<HashMap<String, CacheEntry>>>;

pub fn new_cache() -> Cache { Arc::new(RwLock::new(HashMap::new())) }

pub async fn review_handler(
    State(cache): State<Cache>,
    Json(req): Json<ReviewRequest>,
) -> (StatusCode, Json<ReviewResponse>) {
    let key = format!("{:x}", Sha256::digest(format!("{}{}", req.code, req.level)));

    // Check cache (1-hour TTL)
    if let Some(entry) = cache.read().unwrap().get(&key) {
        if entry.created.elapsed() < Duration::from_secs(3600) {
            return (StatusCode::OK, Json(entry.response.clone()));
        }
    }

    let clippy_output = tokio::task::spawn_blocking({
        let code = req.code.clone();
        move || run_clippy(&code)
    }).await.unwrap_or_default();

    let response = review(&req.code, &req.level, &clippy_output).await;
    cache.write().unwrap().insert(key, CacheEntry { response: response.clone(), created: Instant::now() });
    (StatusCode::OK, Json(response))
}
```

---

## Step 5 — Main

```rust
// src/main.rs
mod model;
mod clippy;
mod ai_review;
mod handler;

use axum::{Router, routing::post};

#[tokio::main]
async fn main() {
    let cache = handler::new_cache();
    let app = Router::new()
        .route("/review", post(handler::review_handler))
        .with_state(cache);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("Code review API at http://localhost:3000");
    axum::serve(listener, app).await.unwrap();
}
```

---

## Step 6 — Test It

```bash
export OPENAI_API_KEY=sk-your-key
cargo run

curl -X POST http://localhost:3000/review \
  -H "Content-Type: application/json" \
  -d '{"code":"fn main(){let x=5;println!(\"{}\",x);}", "level":"beginner"}'
```

---

## Stretch Goals

- [ ] Add rate limiting: 10 reviews/minute per IP
- [ ] Serve a simple HTML form at `GET /` so users can paste code in a browser
- [ ] Add support for reviewing multiple files uploaded as a ZIP
- [ ] Add a webhook to post reviews to a Slack channel

---

[Back to All Projects](../index.md){ .md-button }


[Next Track: Enterprise Projects](../06-enterprise-apps/ecommerce-api.md){ .md-button .md-button--primary }
