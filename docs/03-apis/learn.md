# Track 3 — APIs: Learn

Track 3 takes you from web server basics to the full API development lifecycle: designing and documenting REST endpoints, serialising complex data, consuming third-party APIs from Rust, and building GraphQL services.

**Prerequisites:** Track 2 — Web Development.

---

## 1. Serde — Serialisation & Deserialisation

`serde` is the backbone of API development in Rust. It derives JSON (and many other formats) serialisers/deserialisers at compile time with zero runtime overhead.

```toml
[dependencies]
serde      = { version = "1", features = ["derive"] }
serde_json = "1"
```

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Product {
    id:          u32,
    name:        String,
    price:       f64,
    in_stock:    bool,
    tags:        Vec<String>,
    description: Option<String>,
}

fn main() {
    let p = Product {
        id: 1,
        name: "Ferris Plushie".into(),
        price: 19.99,
        in_stock: true,
        tags: vec!["rust".into(), "merch".into()],
        description: None,
    };

    let json = serde_json::to_string_pretty(&p).unwrap();
    println!("{json}");

    let back: Product = serde_json::from_str(&json).unwrap();
    println!("{back:?}");
}
```

### Serde attributes

```rust
#[derive(Serialize, Deserialize)]
struct Event {
    #[serde(rename = "event_type")]
    kind: String,

    #[serde(skip_serializing_if = "Option::is_none")]
    payload: Option<String>,

    #[serde(default)]
    retries: u32,

    #[serde(alias = "ts", alias = "timestamp")]
    created_at: u64,
}
```

---

## 2. REST API Design Patterns

A well-designed REST API follows these conventions:

| Resource action | Method | Path | Status |
|-----------------|--------|------|--------|
| List all | `GET` | `/products` | 200 |
| Get one | `GET` | `/products/:id` | 200 / 404 |
| Create | `POST` | `/products` | 201 |
| Full update | `PUT` | `/products/:id` | 200 / 404 |
| Partial update | `PATCH` | `/products/:id` | 200 / 404 |
| Delete | `DELETE` | `/products/:id` | 204 / 404 |

### Pagination

```rust
#[derive(Deserialize)]
struct PageParams {
    #[serde(default = "default_page")]
    page: u32,
    #[serde(default = "default_per_page")]
    per_page: u32,
}
fn default_page() -> u32 { 1 }
fn default_per_page() -> u32 { 20 }

#[derive(Serialize)]
struct Paginated<T> {
    data:        Vec<T>,
    page:        u32,
    per_page:    u32,
    total:       usize,
    total_pages: u32,
}
```

---

## 3. HTTP Client with `reqwest`

`reqwest` is the standard async HTTP client for Rust.

```toml
[dependencies]
reqwest = { version = "0.12", features = ["json"] }
tokio   = { version = "1",   features = ["full"] }
```

```rust
use reqwest::Client;
use serde::Deserialize;

#[derive(Deserialize, Debug)]
struct Post { id: u32, title: String, body: String }

#[tokio::main]
async fn main() -> Result<(), reqwest::Error> {
    let client = Client::new();

    // GET with JSON response
    let posts: Vec<Post> = client
        .get("https://jsonplaceholder.typicode.com/posts")
        .send()
        .await?
        .json()
        .await?;
    println!("Got {} posts", posts.len());

    // POST with JSON body
    let new_post: serde_json::Value = client
        .post("https://jsonplaceholder.typicode.com/posts")
        .json(&serde_json::json!({ "title": "Rust", "body": "🦀", "userId": 1 }))
        .send()
        .await?
        .json()
        .await?;
    println!("{new_post}");
    Ok(())
}
```

### Headers & auth

```rust
use reqwest::header::{AUTHORIZATION, CONTENT_TYPE};

let response = client
    .get("https://api.example.com/data")
    .header(AUTHORIZATION, "Bearer my-token")
    .header(CONTENT_TYPE, "application/json")
    .send()
    .await?;
```

### Reusing a client with connection pooling

```rust
// Build once, share via Arc or AppState
let client = Client::builder()
    .timeout(std::time::Duration::from_secs(10))
    .user_agent("my-app/1.0")
    .build()?;
```

---

## 4. Building a Complete REST API

Putting it all together — a complete products API:

```rust
use axum::{Router, routing::{get, post, put, delete}, extract::{Path, State, Query}, Json, http::StatusCode, response::IntoResponse};
use serde::{Deserialize, Serialize};
use std::sync::{Arc, RwLock};
use std::collections::HashMap;

#[derive(Clone, Serialize, Deserialize)]
struct Product { id: u32, name: String, price: f64 }

type Db = Arc<RwLock<HashMap<u32, Product>>>;

async fn list(State(db): State<Db>) -> Json<Vec<Product>> {
    Json(db.read().unwrap().values().cloned().collect())
}

async fn create(State(db): State<Db>, Json(p): Json<Product>)
    -> (StatusCode, Json<Product>)
{
    db.write().unwrap().insert(p.id, p.clone());
    (StatusCode::CREATED, Json(p))
}

async fn get_one(State(db): State<Db>, Path(id): Path<u32>)
    -> impl IntoResponse
{
    match db.read().unwrap().get(&id).cloned() {
        Some(p) => (StatusCode::OK, Json(serde_json::to_value(p).unwrap())),
        None    => (StatusCode::NOT_FOUND, Json(serde_json::json!({"error":"not found"}))),
    }
}

#[tokio::main]
async fn main() {
    let db: Db = Arc::new(RwLock::new(HashMap::new()));
    let app = Router::new()
        .route("/products",     get(list).post(create))
        .route("/products/:id", get(get_one))
        .with_state(db);
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

---

## 5. OpenAPI Documentation with `utoipa`

`utoipa` generates an OpenAPI 3.x spec from your Rust types and handlers.

```toml
[dependencies]
utoipa      = { version = "4", features = ["axum_extras"] }
utoipa-swagger-ui = { version = "7", features = ["axum"] }
```

```rust
use utoipa::{OpenApi, ToSchema};
use utoipa_swagger_ui::SwaggerUi;

#[derive(Serialize, Deserialize, ToSchema)]
struct Product { id: u32, name: String, price: f64 }

#[derive(OpenApi)]
#[openapi(
    paths(list_products, get_product),
    components(schemas(Product)),
    tags((name = "products", description = "Product management"))
)]
struct ApiDoc;

// Serve the Swagger UI at /swagger-ui
let app = Router::new()
    .merge(SwaggerUi::new("/swagger-ui").url("/api-docs/openapi.json", ApiDoc::openapi()));
```

---

## 6. GraphQL with `async-graphql`

```toml
[dependencies]
async-graphql      = "7"
async-graphql-axum = "7"
```

```rust
use async_graphql::{Object, Schema, SimpleObject, EmptyMutation, EmptySubscription};
use async_graphql_axum::{GraphQLRequest, GraphQLResponse};

#[derive(SimpleObject)]
struct User { id: u32, name: String }

struct Query;

#[Object]
impl Query {
    async fn user(&self, id: u32) -> User {
        User { id, name: "Ferris".into() }
    }
    async fn users(&self) -> Vec<User> {
        vec![User { id: 1, name: "Alice".into() }, User { id: 2, name: "Bob".into() }]
    }
}

type AppSchema = Schema<Query, EmptyMutation, EmptySubscription>;

async fn graphql_handler(schema: axum::Extension<AppSchema>, req: GraphQLRequest)
    -> GraphQLResponse
{
    schema.execute(req.into_inner()).await.into()
}
```

---

## 7. API Versioning

Two common strategies in Axum:

```rust
// URL path versioning
let app = Router::new()
    .nest("/v1", v1_router())
    .nest("/v2", v2_router());

// Header versioning (Accept: application/vnd.api+json;version=2)
// Handled in middleware that inspects the Accept header.
```

---

## 8. Authentication — JWT

```toml
[dependencies]
jsonwebtoken = "9"
```

```rust
use jsonwebtoken::{encode, decode, Header, Validation, EncodingKey, DecodingKey};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Claims { sub: String, exp: usize }

fn create_token(user_id: &str, secret: &[u8]) -> String {
    let claims = Claims {
        sub: user_id.to_owned(),
        exp: (chrono::Utc::now() + chrono::Duration::hours(24)).timestamp() as usize,
    };
    encode(&Header::default(), &claims, &EncodingKey::from_secret(secret)).unwrap()
}

fn verify_token(token: &str, secret: &[u8]) -> Result<Claims, jsonwebtoken::errors::Error> {
    decode::<Claims>(token, &DecodingKey::from_secret(secret), &Validation::default())
        .map(|data| data.claims)
}
```

---

## What's Next?

[Go to Practice :fontawesome-solid-dumbbell:](practice.md){ .md-button .md-button--primary }
[Jump to Challenges :fontawesome-solid-trophy:](challenges.md){ .md-button }
