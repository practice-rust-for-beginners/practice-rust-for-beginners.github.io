# Project: Personal Blog Engine

**Track:** 🌐 Web Development &nbsp;|&nbsp; **Difficulty:** 🟡 Intermediate &nbsp;|&nbsp; **Crates:** axum, tokio, askama, pulldown-cmark, serde

---

## What You'll Build

A server-side rendered blog with Markdown post rendering, tag filtering, and an RSS feed — no JavaScript framework, no database. Posts are plain `.md` files on disk.

```text
GET /                → home page, 5 latest posts
GET /posts           → all posts, filterable by tag
GET /posts/:slug     → single post rendered from Markdown
GET /tags/:tag       → posts with a specific tag
GET /feed.xml        → RSS 2.0 feed
```

---

## Tech Stack

| Component | Crate |
|-----------|-------|
| HTTP server | `axum 0.7` |
| Async runtime | `tokio 1` |
| HTML templates | `askama 0.12` + `askama_axum` |
| Markdown → HTML | `pulldown-cmark 0.11` |
| Front-matter parsing | `gray_matter 0.2` |
| Serialisation | `serde 1` |

```toml
[dependencies]
axum          = { version = "0.7", features = ["macros"] }
tokio         = { version = "1",   features = ["full"] }
askama        = "0.12"
askama_axum   = "0.4"
pulldown-cmark = "0.11"
gray_matter   = "0.2"
serde         = { version = "1", features = ["derive"] }
tower-http    = { version = "0.5", features = ["fs"] }
chrono        = { version = "0.4", features = ["serde"] }
```

---

## Step 1 — Post Structure & Loading

Each post is a `.md` file with YAML front-matter:

```markdown
---
title: "Hello, Rust!"
date: 2024-06-01
tags: [rust, beginners]
summary: "My first Rust blog post."
---

## Introduction

Rust is a systems programming language...
```

```rust
// src/posts.rs
use serde::Deserialize;
use chrono::NaiveDate;
use std::{fs, path::Path};
use gray_matter::{Matter, engine::YAML};
use pulldown_cmark::{Parser, Options, html};

#[derive(Debug, Clone, Deserialize)]
pub struct FrontMatter {
    pub title:   String,
    pub date:    NaiveDate,
    pub tags:    Vec<String>,
    pub summary: String,
}

#[derive(Debug, Clone)]
pub struct Post {
    pub slug:    String,
    pub meta:    FrontMatter,
    pub html:    String,
}

pub fn load_all(posts_dir: &str) -> Vec<Post> {
    let mut posts = Vec::new();
    let Ok(entries) = fs::read_dir(posts_dir) else { return posts; };

    for entry in entries.flatten() {
        let path = entry.path();
        if path.extension().and_then(|e| e.to_str()) != Some("md") { continue; }

        let slug = path.file_stem().unwrap().to_string_lossy().to_string();
        let raw  = fs::read_to_string(&path).unwrap();
        let matter = Matter::<YAML>::new();
        let parsed = matter.parse(&raw);
        let meta: FrontMatter = parsed.data.unwrap().deserialize().unwrap();
        let html_body = md_to_html(&parsed.content);

        posts.push(Post { slug, meta, html: html_body });
    }
    posts.sort_by(|a, b| b.meta.date.cmp(&a.meta.date));
    posts
}

fn md_to_html(md: &str) -> String {
    let parser = Parser::new_ext(md, Options::all());
    let mut html_output = String::new();
    html::push_html(&mut html_output, parser);
    html_output
}
```

---

## Step 2 — Askama Templates

```text
templates/
  base.html      ← shared layout
  index.html     ← home page
  post.html      ← single post
  posts.html     ← post list
```

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>{% block title %}My Rust Blog{% endblock %}</title>
  <style>
    body { font-family: system-ui; max-width: 760px; margin: 2rem auto; padding: 0 1rem; }
    nav a { margin-right: 1rem; }
    .tag { background: #e5e7eb; border-radius: 4px; padding: 2px 8px; font-size: 0.8rem; }
  </style>
</head>
<body>
  <nav><a href="/">Home</a><a href="/posts">All Posts</a></nav>
  <hr>
  {% block content %}{% endblock %}
</body>
</html>
```

```html
<!-- templates/post.html -->
{% extends "base.html" %}
{% block title %}{{ post.meta.title }}{% endblock %}
{% block content %}
  <h1>{{ post.meta.title }}</h1>
  <p>{{ post.meta.date }} &nbsp;
    {% for tag in post.meta.tags %}
      <a href="/tags/{{ tag }}" class="tag">{{ tag }}</a>
    {% endfor %}
  </p>
  {{ post.html|safe }}
{% endblock %}
```

---

## Step 3 — Handlers & Router

```rust
// src/handlers.rs
use axum::{extract::{Path, Query, State}, response::Html};
use askama::Template;
use askama_axum::IntoResponse;
use crate::posts::Post;
use serde::Deserialize;
use std::sync::Arc;

pub type AppState = Arc<Vec<Post>>;

#[derive(Template)]
#[template(path = "index.html")]
struct IndexTemplate<'a> { posts: &'a [Post] }

#[derive(Template)]
#[template(path = "post.html")]
struct PostTemplate<'a> { post: &'a Post }

pub async fn index(State(posts): State<AppState>) -> impl IntoResponse {
    IndexTemplate { posts: &posts[..5.min(posts.len())] }
}

pub async fn post(State(posts): State<AppState>, Path(slug): Path<String>)
    -> impl IntoResponse
{
    match posts.iter().find(|p| p.slug == slug) {
        Some(post) => PostTemplate { post }.into_response(),
        None       => axum::http::StatusCode::NOT_FOUND.into_response(),
    }
}

pub async fn rss_feed(State(posts): State<AppState>) -> impl axum::response::IntoResponse {
    let mut items = String::new();
    for p in posts.iter().take(10) {
        items.push_str(&format!(
            "<item><title>{}</title><link>https://myblog.com/posts/{}</link>\
             <description>{}</description></item>",
            p.meta.title, p.slug, p.meta.summary
        ));
    }
    let xml = format!(
        r#"<?xml version="1.0"?><rss version="2.0"><channel>
        <title>My Rust Blog</title>{items}</channel></rss>"#
    );
    ([("content-type", "application/rss+xml")], xml)
}
```

```rust
// src/main.rs
mod posts;
mod handlers;

use axum::{Router, routing::get};
use handlers::AppState;
use std::sync::Arc;
use tower_http::services::ServeDir;

#[tokio::main]
async fn main() {
    let posts = Arc::new(posts::load_all("./posts"));
    println!("Loaded {} posts", posts.len());

    let app = Router::new()
        .route("/",           get(handlers::index))
        .route("/posts/:slug",get(handlers::post))
        .route("/feed.xml",   get(handlers::rss_feed))
        .nest_service("/static", ServeDir::new("static"))
        .with_state(posts);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("Serving at http://localhost:3000");
    axum::serve(listener, app).await.unwrap();
}
```

---

## Step 4 — Create Some Posts

```bash
mkdir posts
cat > posts/hello-rust.md << 'EOF'
---
title: "Hello, Rust!"
date: 2024-06-01
tags: [rust, beginners]
summary: "My first Rust blog post."
---

## Introduction

Rust is a systems programming language focused on speed and safety.
EOF
```

```bash
cargo run
# Visit http://localhost:3000
```

---

## Stretch Goals

- [ ] Add pagination (`/posts?page=2`)
- [ ] Add full-text search (scan post HTML on the server, return matching excerpts)
- [ ] Add a sitemap at `/sitemap.xml`
- [ ] Watch `./posts/` for changes and hot-reload posts without restarting

---

[Back to All Projects](../index.md){ .md-button }


[Next Project: Link Shortener](link-shortener.md){ .md-button .md-button--primary }
