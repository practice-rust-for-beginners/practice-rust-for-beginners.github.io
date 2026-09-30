# Track 2 — Web Development: Learn

In Track 1 you mastered Rust's core language. Now you'll use that foundation to build real HTTP servers and web applications. Rust's async story — powered by **Tokio** — combined with the ergonomic **Axum** framework makes it one of the fastest and safest ways to write web services.

**Prerequisites:** Complete Track 1 — Fundamentals.

---

## 1. The Async Model — Tokio

Web servers handle many concurrent connections. Rust's answer is **async/await**, built on lightweight tasks scheduled by a runtime. The most widely used runtime is **Tokio**.

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
axum  = "0.7"
```

```rust
// The simplest possible async program
#[tokio::main]
async fn main() {
    println!("Running on Tokio!");
}
```

!!! info "async/await"
    - `async fn` returns a `Future` — it describes work to do, not work in progress.
    - `.await` drives the future to completion, yielding control back to the runtime while waiting.
    - Unlike threads, thousands of async tasks can run on a small thread pool with minimal overhead.

---

## 2. Hello, Axum!

```rust
use axum::{Router, routing::get};
use tokio::net::TcpListener;

async fn hello() -> &'static str {
    "Hello, Axum!"
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(hello));
    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("Listening on http://localhost:3000");
    axum::serve(listener, app).await.unwrap();
}
```

Run it and visit `http://localhost:3000` — you'll see `Hello, Axum!`.

---

## 3. Routing

Axum routes map HTTP methods + paths to handler functions.

```rust
use axum::{Router, routing::{get, post, put, delete}};

let app = Router::new()
    .route("/",             get(index))
    .route("/users",        get(list_users).post(create_user))
    .route("/users/:id",    get(get_user).put(update_user).delete(delete_user));
```

### Path parameters

```rust
use axum::extract::Path;

async fn get_user(Path(id): Path<u32>) -> String {
    format!("User #{id}")
}
```

### Query parameters

```rust
use axum::extract::Query;
use serde::Deserialize;

#[derive(Deserialize)]
struct Pagination { page: u32, per_page: u32 }

async fn list_users(Query(params): Query<Pagination>) -> String {
    format!("Page {} ({} per page)", params.page, params.per_page)
}
```

---

## 4. JSON Request & Response

Add `serde` and `axum`'s JSON extractor:

```toml
[dependencies]
axum  = { version = "0.7", features = ["json"] }
serde = { version = "1",   features = ["derive"] }
```

```rust
use axum::{Json, extract::Path};
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
struct User { id: u32, name: String, email: String }

// POST /users — read JSON body
async fn create_user(Json(user): Json<User>) -> Json<User> {
    Json(user) // echo back
}

// GET /users/:id — return JSON
async fn get_user(Path(id): Path<u32>) -> Json<User> {
    Json(User { id, name: "Ferris".into(), email: "ferris@rust.org".into() })
}
```

---

## 5. State — Sharing Data Between Handlers

Most real apps need shared state (e.g. a database pool). Axum uses the `State` extractor:

```rust
use axum::{Router, routing::get, extract::State};
use std::sync::Arc;
use tokio::sync::RwLock;

#[derive(Clone)]
struct AppState {
    counter: Arc<RwLock<u64>>,
}

async fn increment(State(state): State<AppState>) -> String {
    let mut n = state.counter.write().await;
    *n += 1;
    format!("Count: {n}")
}

#[tokio::main]
async fn main() {
    let state = AppState { counter: Arc::new(RwLock::new(0)) };
    let app = Router::new()
        .route("/increment", get(increment))
        .with_state(state);
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

!!! tip "Arc vs Mutex vs RwLock"
    - `Arc<T>` — shared ownership across threads (atomic reference counting).
    - `Mutex<T>` — exclusive mutable access.
    - `RwLock<T>` — multiple readers OR one writer; better for read-heavy state.

---

## 6. HTTP Status Codes & Error Responses

Axum handlers can return `(StatusCode, impl IntoResponse)`:

```rust
use axum::http::StatusCode;
use axum::{Json, response::IntoResponse};

async fn not_found_example() -> impl IntoResponse {
    (StatusCode::NOT_FOUND, Json(serde_json::json!({
        "error": "User not found",
        "code": 404
    })))
}
```

### Custom error type

```rust
use axum::{response::{IntoResponse, Response}, http::StatusCode, Json};

enum AppError {
    NotFound(String),
    BadRequest(String),
    Internal(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, msg) = match self {
            AppError::NotFound(m)    => (StatusCode::NOT_FOUND, m),
            AppError::BadRequest(m)  => (StatusCode::BAD_REQUEST, m),
            AppError::Internal(m)    => (StatusCode::INTERNAL_SERVER_ERROR, m),
        };
        (status, Json(serde_json::json!({ "error": msg }))).into_response()
    }
}

async fn handler() -> Result<Json<serde_json::Value>, AppError> {
    Err(AppError::NotFound("User not found".into()))
}
```

---

## 7. Middleware with Tower

Axum is built on **Tower**, a composable service abstraction. Tower middleware runs around every request.

```toml
[dependencies]
tower-http = { version = "0.5", features = ["cors", "trace", "compression-gzip"] }
tracing-subscriber = "0.3"
```

```rust
use tower_http::{cors::CorsLayer, trace::TraceLayer, compression::CompressionLayer};
use axum::Router;

let app = Router::new()
    .route("/", get(handler))
    .layer(TraceLayer::new_for_http())
    .layer(CorsLayer::permissive())
    .layer(CompressionLayer::new());
```

---

## 8. HTML Templating with Askama

For server-side rendered HTML, **Askama** compiles Jinja-style templates at build time.

```toml
[dependencies]
askama      = "0.12"
askama_axum = "0.4"
```

```html
<!-- templates/index.html -->
<!DOCTYPE html>
<html>
<body>
  <h1>Hello, {{ name }}!</h1>
  <ul>
    {% for item in items %}
      <li>{{ item }}</li>
    {% endfor %}
  </ul>
</body>
</html>
```

```rust
use askama::Template;
use askama_axum::IntoResponse;

#[derive(Template)]
#[template(path = "index.html")]
struct IndexTemplate<'a> {
    name:  &'a str,
    items: Vec<String>,
}

async fn index() -> impl IntoResponse {
    IndexTemplate {
        name: "Ferris",
        items: vec!["Learn Rust".into(), "Build stuff".into()],
    }
}
```

---

## 9. Serving Static Files

```toml
[dependencies]
tower-http = { version = "0.5", features = ["fs"] }
```

```rust
use tower_http::services::ServeDir;

let app = Router::new()
    .nest_service("/static", ServeDir::new("static"));
```

Place files in `./static/` and they'll be served at `/static/filename`.

---

## 10. Form Handling

```toml
[dependencies]
axum = { version = "0.7", features = ["form"] }
```

```rust
use axum::Form;
use serde::Deserialize;

#[derive(Deserialize)]
struct LoginForm { username: String, password: String }

async fn login(Form(form): Form<LoginForm>) -> String {
    format!("Welcome, {}!", form.username)
}
```

---

## 11. WebSockets

```rust
use axum::{Router, routing::get, extract::WebSocketUpgrade, response::IntoResponse};
use axum::extract::ws::{WebSocket, Message};

async fn ws_handler(ws: WebSocketUpgrade) -> impl IntoResponse {
    ws.on_upgrade(handle_socket)
}

async fn handle_socket(mut socket: WebSocket) {
    while let Some(Ok(msg)) = socket.recv().await {
        if let Message::Text(text) = msg {
            let _ = socket.send(Message::Text(format!("Echo: {text}"))).await;
        }
    }
}
```

---

## What's Next?

[Go to Practice](practice.md){ .md-button .md-button--primary }

[Jump to Challenges](challenges.md){ .md-button }
