# Track 2 — Web Development: Practice

Exercises to build up your Axum skills hands-on. Each exercise is a standalone `cargo` project.

!!! tip "Setup for all exercises"
    ```toml
    # Cargo.toml
    [dependencies]
    axum     = { version = "0.7", features = ["json", "form"] }
    tokio    = { version = "1",   features = ["full"] }
    serde    = { version = "1",   features = ["derive"] }
    serde_json = "1"
    tower-http = { version = "0.5", features = ["cors", "trace"] }
    ```

---

## Exercise 1 — Static Routes

Build a server with these routes:

| Route | Response |
|-------|----------|
| `GET /` | `"Welcome to my Axum server!"` |
| `GET /ping` | `"pong"` |
| `GET /health` | JSON `{"status": "ok"}` |

Test with `curl`:
```bash
curl http://localhost:3000/
curl http://localhost:3000/ping
curl http://localhost:3000/health
```

---

## Exercise 2 — Path & Query Extraction

Create an endpoint `GET /greet/:name?loud=true|false` that:
- Greets the person by name.
- If `?loud=true`, returns the greeting in uppercase.

```bash
curl "http://localhost:3000/greet/ferris"           # Hello, ferris!
curl "http://localhost:3000/greet/ferris?loud=true" # HELLO, FERRIS!
```

??? example "Solution sketch"
    ```rust
    #[derive(Deserialize)]
    struct LoudQuery { loud: Option<bool> }

    async fn greet(
        Path(name): Path<String>,
        Query(q): Query<LoudQuery>,
    ) -> String {
        let greeting = format!("Hello, {name}!");
        if q.loud.unwrap_or(false) { greeting.to_uppercase() } else { greeting }
    }
    ```

---

## Exercise 3 — In-memory CRUD

Build a simple in-memory `users` resource. Use `Arc<RwLock<Vec<User>>>` as state.

```
POST   /users           — create, returns 201 + JSON body
GET    /users           — list all
GET    /users/:id       — get one (404 if not found)
PUT    /users/:id       — update name/email (404 if not found)
DELETE /users/:id       — delete (204 no content)
```

```rust
#[derive(Clone, Serialize, Deserialize)]
struct User { id: u32, name: String, email: String }
```

Test with:
```bash
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"id":1,"name":"Alice","email":"alice@example.com"}'

curl http://localhost:3000/users
curl http://localhost:3000/users/1
curl -X DELETE http://localhost:3000/users/1
```

---

## Exercise 4 — Custom Error Type

Extend Exercise 3 so that all error responses are consistent JSON:

```json
{ "error": "User with id 99 not found", "code": 404 }
```

Define an `AppError` enum with `IntoResponse` so every handler can `return Err(AppError::NotFound(...))`.

---

## Exercise 5 — Request Validation Middleware

Add a middleware layer that:
1. Reads a request header `X-API-Key`.
2. Rejects requests with status `401` if the key is not `"secret123"`.
3. Allows all requests through if the key matches.

!!! tip "Tower middleware"
    Use `axum::middleware::from_fn` for a simple async function middleware.

```rust
use axum::middleware;

let app = Router::new()
    .route("/protected", get(protected_handler))
    .layer(middleware::from_fn(api_key_auth));
```

---

## Exercise 6 — File Upload Endpoint

Create `POST /upload` that:
1. Accepts a `multipart/form-data` request with a field named `file`.
2. Returns JSON with the file name and byte size.

```toml
[dependencies]
axum-multipart = "0.6"  # or use axum's built-in multipart
```

---

## Exercise 7 — HTML Page with Askama

Build a mini website with two pages:
- `GET /` — a home page listing 3 blog posts (title + date).
- `GET /posts/:id` — a detail page showing the full post.

Store posts in a `Vec<Post>` in `AppState`. Render HTML with Askama templates.

---

## Exercise 8 — WebSocket Echo Server

Build a WebSocket endpoint that:
1. Accepts a WS connection at `ws://localhost:3000/ws`.
2. Echoes every text message back prefixed with `"Echo: "`.
3. Closes the connection if the client sends `"quit"`.

Test with a tool like `websocat`:
```bash
websocat ws://localhost:3000/ws
```

---

[Go to Challenges :fontawesome-solid-trophy:](challenges.md){ .md-button .md-button--primary }
[Back to Learn :fontawesome-solid-book-open:](learn.md){ .md-button }
