# Project: Serverless URL Shortener

**Track:** ☁️ Cloud &nbsp;|&nbsp; **Difficulty:** 🟡 Intermediate &nbsp;|&nbsp; **Crates:** lambda_http, aws-sdk-dynamodb, serde, tokio

---

## What You'll Build

Rebuild the Link Shortener project as an AWS Lambda function. Each request is handled by a cold-starting Rust binary with millisecond boot times. DynamoDB stores the short links.

```text
POST https://{api-gw}.execute-api.us-east-1.amazonaws.com/shorten
→ {"short": "https://short.example.com/r/aB3xY"}

GET  https://{api-gw}.execute-api.us-east-1.amazonaws.com/r/aB3xY
→ 302 → https://www.rust-lang.org
```

---

## Tech Stack

```toml
[dependencies]
lambda_http        = "0.11"
aws-config         = "1"
aws-sdk-dynamodb   = "1"
serde              = { version = "1", features = ["derive"] }
serde_json         = "1"
tokio              = { version = "1", features = ["full"] }
rand               = "0.8"
```

```toml
[[bin]]
name = "bootstrap"   # Lambda requires the binary to be named "bootstrap"
path = "src/main.rs"
```

---

## Step 1 — DynamoDB Table

Create the table via AWS CLI:

```bash
aws dynamodb create-table \
  --table-name short-links \
  --attribute-definitions AttributeName=code,AttributeType=S \
  --key-schema AttributeName=code,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

---

## Step 2 — DynamoDB Helper

```rust
// src/db.rs
use aws_sdk_dynamodb::{Client, types::AttributeValue};
use std::collections::HashMap;

pub struct LinkStore { client: Client, table: String }

impl LinkStore {
    pub fn new(client: Client, table: impl Into<String>) -> Self {
        Self { client, table: table.into() }
    }

    pub async fn put(&self, code: &str, url: &str) -> Result<(), aws_sdk_dynamodb::Error> {
        self.client.put_item()
            .table_name(&self.table)
            .item("code", AttributeValue::S(code.to_string()))
            .item("url",  AttributeValue::S(url.to_string()))
            .send().await?;
        Ok(())
    }

    pub async fn get(&self, code: &str) -> Result<Option<String>, aws_sdk_dynamodb::Error> {
        let resp = self.client.get_item()
            .table_name(&self.table)
            .key("code", AttributeValue::S(code.to_string()))
            .send().await?;
        Ok(resp.item()
            .and_then(|item| item.get("url"))
            .and_then(|v| v.as_s().ok())
            .map(|s| s.clone()))
    }
}
```

---

## Step 3 — Lambda Handler

```rust
// src/main.rs
mod db;

use lambda_http::{run, service_fn, Body, Error, Request, Response, http::{StatusCode, header}};
use aws_sdk_dynamodb::Client;
use db::LinkStore;
use rand::Rng;

fn generate_code() -> String {
    let mut rng = rand::thread_rng();
    (0..6).map(|_| {
        "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
            .chars().nth(rng.gen_range(0..62)).unwrap()
    }).collect()
}

async fn handler(store: &LinkStore, event: Request) -> Result<Response<Body>, Error> {
    let path = event.uri().path();

    if path == "/shorten" && event.method() == lambda_http::http::Method::POST {
        let body = std::str::from_utf8(event.body()).unwrap_or("");
        let parsed: serde_json::Value = serde_json::from_str(body)?;
        let url = parsed["url"].as_str().unwrap_or("");
        let code = generate_code();
        store.put(&code, url).await?;
        let short = format!("https://short.example.com/r/{code}");
        return Ok(Response::builder()
            .status(201)
            .header("content-type", "application/json")
            .body(Body::Text(serde_json::json!({"short": short, "code": code}).to_string()))?);
    }

    if path.starts_with("/r/") {
        let code = &path[3..];
        if let Some(url) = store.get(code).await? {
            return Ok(Response::builder()
                .status(StatusCode::FOUND)
                .header(header::LOCATION, url)
                .body(Body::Empty)?);
        }
        return Ok(Response::builder().status(404).body(Body::Text("Not found".into()))?);
    }

    Ok(Response::builder().status(400).body(Body::Text("Bad request".into()))?)
}

#[tokio::main]
async fn main() -> Result<(), Error> {
    let config = aws_config::load_from_env().await;
    let client = Client::new(&config);
    let table  = std::env::var("TABLE_NAME").unwrap_or("short-links".into());
    let store  = LinkStore::new(client, table);

    run(service_fn(|req| handler(&store, req))).await
}
```

---

## Step 4 — Build & Deploy

```bash
cargo install cargo-lambda

# Local testing
TABLE_NAME=short-links cargo lambda watch

# In another terminal
curl -X POST http://localhost:9000/lambda-url/bootstrap/ \
  -d '{"url":"https://rust-lang.org"}'

# Build for Lambda (ARM — cheaper + faster)
cargo lambda build --release --arm64

# Deploy
cargo lambda deploy \
  --binary-name bootstrap \
  --iam-role arn:aws:iam::123456789012:role/lambda-exec-role \
  --env-vars TABLE_NAME=short-links \
  bootstrap
```

---

## Stretch Goals

- [ ] Add an API Gateway trigger with a custom domain
- [ ] Use DynamoDB TTL attribute to auto-expire short links
- [ ] Add click counting with a DynamoDB atomic counter
- [ ] Add AWS X-Ray tracing with `aws-xray-sdk`

---

[Back to All Projects](../index.md){ .md-button }


[Next Project: CI/CD Pipeline](cicd-pipeline.md){ .md-button .md-button--primary }
