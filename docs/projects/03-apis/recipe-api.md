# Project: Recipe API

**Track:** 🔌 APIs &nbsp;|&nbsp; **Difficulty:** 🟡 Intermediate &nbsp;|&nbsp; **Crates:** axum, serde, utoipa, tokio

---

## What You'll Build

A fully documented REST API for a recipe collection with search, tag filtering, and an auto-generated Swagger UI. The data lives in memory — no database needed.

```text
GET  /recipes                    → list all (filterable: ?tag=vegan&q=soup)
GET  /recipes/:id                → get one recipe
POST /recipes                    → create recipe
PUT  /recipes/:id                → update recipe
DELETE /recipes/:id              → delete recipe
GET  /swagger-ui                 → interactive API docs
```

---

## Tech Stack

```toml
[dependencies]
axum        = { version = "0.7", features = ["json"] }
tokio       = { version = "1",   features = ["full"] }
serde       = { version = "1",   features = ["derive"] }
serde_json  = "1"
utoipa      = { version = "4",   features = ["axum_extras"] }
utoipa-swagger-ui = { version = "7", features = ["axum"] }
uuid        = { version = "1",   features = ["v4", "serde"] }
```

---

## Step 1 — Data Model with OpenAPI Annotations

```rust
// src/model.rs
use serde::{Deserialize, Serialize};
use utoipa::ToSchema;
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize, ToSchema)]
pub struct Recipe {
    pub id:           Uuid,
    pub title:        String,
    pub description:  String,
    pub ingredients:  Vec<String>,
    pub instructions: Vec<String>,
    pub tags:         Vec<String>,
    pub servings:     u8,
    pub prep_minutes: u32,
}

#[derive(Debug, Deserialize, ToSchema)]
pub struct CreateRecipe {
    pub title:        String,
    pub description:  String,
    pub ingredients:  Vec<String>,
    pub instructions: Vec<String>,
    pub tags:         Vec<String>,
    pub servings:     u8,
    pub prep_minutes: u32,
}

#[derive(Debug, Deserialize)]
pub struct RecipeQuery {
    pub q:   Option<String>,
    pub tag: Option<String>,
}
```

---

## Step 2 — Store & Seed Data

```rust
// src/store.rs
use std::collections::HashMap;
use std::sync::{Arc, RwLock};
use uuid::Uuid;
use crate::model::Recipe;

pub type RecipeStore = Arc<RwLock<HashMap<Uuid, Recipe>>>;

pub fn new_store() -> RecipeStore {
    let mut map = HashMap::new();

    let pasta = Recipe {
        id: Uuid::new_v4(),
        title: "Spaghetti Bolognese".into(),
        description: "Classic Italian meat sauce pasta.".into(),
        ingredients: vec!["spaghetti".into(), "beef mince".into(), "tomato passata".into()],
        instructions: vec!["Brown the mince.".into(), "Add passata, simmer 20 min.".into(), "Cook pasta.".into()],
        tags: vec!["italian".into(), "pasta".into()],
        servings: 4,
        prep_minutes: 30,
    };
    map.insert(pasta.id, pasta);

    Arc::new(RwLock::new(map))
}
```

---

## Step 3 — Handlers with utoipa Annotations

```rust
// src/handlers.rs
use axum::{extract::{Path, Query, State}, Json, http::StatusCode, response::IntoResponse};
use utoipa::OpenApi;
use uuid::Uuid;
use crate::{model::{Recipe, CreateRecipe, RecipeQuery}, store::RecipeStore};

#[utoipa::path(get, path = "/recipes",
    params(("q" = Option<String>, Query, description = "Search term"),
           ("tag" = Option<String>, Query, description = "Filter by tag")),
    responses((status = 200, body = Vec<Recipe>)))]
pub async fn list(State(store): State<RecipeStore>, Query(q): Query<RecipeQuery>)
    -> Json<Vec<Recipe>>
{
    let recipes: Vec<Recipe> = store.read().unwrap().values()
        .filter(|r| {
            q.q.as_deref().map_or(true, |s| r.title.to_lowercase().contains(s))
            && q.tag.as_deref().map_or(true, |t| r.tags.iter().any(|rt| rt == t))
        })
        .cloned().collect();
    Json(recipes)
}

#[utoipa::path(post, path = "/recipes",
    request_body = CreateRecipe,
    responses((status = 201, body = Recipe)))]
pub async fn create(State(store): State<RecipeStore>, Json(body): Json<CreateRecipe>)
    -> (StatusCode, Json<Recipe>)
{
    let recipe = Recipe {
        id: Uuid::new_v4(),
        title: body.title, description: body.description,
        ingredients: body.ingredients, instructions: body.instructions,
        tags: body.tags, servings: body.servings, prep_minutes: body.prep_minutes,
    };
    store.write().unwrap().insert(recipe.id, recipe.clone());
    (StatusCode::CREATED, Json(recipe))
}

#[utoipa::path(get, path = "/recipes/{id}",
    params(("id" = Uuid, Path, description = "Recipe ID")),
    responses((status = 200, body = Recipe), (status = 404, description = "Not found")))]
pub async fn get_one(State(store): State<RecipeStore>, Path(id): Path<Uuid>)
    -> impl IntoResponse
{
    match store.read().unwrap().get(&id).cloned() {
        Some(r) => Json(r).into_response(),
        None    => StatusCode::NOT_FOUND.into_response(),
    }
}

pub async fn delete(State(store): State<RecipeStore>, Path(id): Path<Uuid>) -> StatusCode {
    if store.write().unwrap().remove(&id).is_some() { StatusCode::NO_CONTENT }
    else { StatusCode::NOT_FOUND }
}
```

---

## Step 4 — OpenAPI Spec + Swagger UI

```rust
// src/main.rs
mod model;
mod store;
mod handlers;

use axum::{Router, routing::{get, post, delete}};
use utoipa::OpenApi;
use utoipa_swagger_ui::SwaggerUi;

#[derive(OpenApi)]
#[openapi(
    paths(handlers::list, handlers::create, handlers::get_one),
    components(schemas(model::Recipe, model::CreateRecipe)),
    tags((name = "recipes", description = "Recipe management API"))
)]
struct ApiDoc;

#[tokio::main]
async fn main() {
    let store = store::new_store();

    let app = Router::new()
        .merge(SwaggerUi::new("/swagger-ui")
            .url("/api-docs/openapi.json", ApiDoc::openapi()))
        .route("/recipes",     get(handlers::list).post(handlers::create))
        .route("/recipes/:id", get(handlers::get_one).delete(handlers::delete))
        .with_state(store);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("API running at http://localhost:3000");
    println!("Swagger UI at http://localhost:3000/swagger-ui");
    axum::serve(listener, app).await.unwrap();
}
```

---

## Step 5 — Run & Explore

```bash
cargo run
# Open http://localhost:3000/swagger-ui in your browser
# Try each endpoint interactively
```

---

## Stretch Goals

- [ ] Add pagination: `?page=1&per_page=10`
- [ ] Add `PATCH /recipes/:id` for partial updates
- [ ] Persist recipes to a JSON file on disk
- [ ] Add `GET /recipes/random` that returns a random recipe

---

[Back to All Projects](../index.md){ .md-button }


[Next Project: GitHub Dashboard](github-dashboard.md){ .md-button .md-button--primary }
