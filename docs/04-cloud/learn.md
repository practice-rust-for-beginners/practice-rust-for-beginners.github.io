# Track 4 — Cloud: Learn

Track 4 covers packaging Rust applications as containers, deploying them to Kubernetes and serverless platforms, and managing configuration and secrets in cloud environments.

**Prerequisites:** Track 3 — APIs.

---

## 1. Containerising Rust with Docker

Docker multi-stage builds are essential for Rust — the build stage is large (the toolchain), but the runtime image can be tiny.

```dockerfile
# Dockerfile
# ── Stage 1: Builder ─────────────────────────────────────────
FROM rust:1.79-slim AS builder
WORKDIR /app

# Cache dependencies separately from source code
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo 'fn main() {}' > src/main.rs
RUN cargo build --release
RUN rm -f target/release/deps/myapp*

# Build the real binary
COPY src ./src
RUN cargo build --release

# ── Stage 2: Runtime ─────────────────────────────────────────
FROM debian:bookworm-slim AS runtime
WORKDIR /app

# Only copy the compiled binary
COPY --from=builder /app/target/release/myapp ./myapp

EXPOSE 3000
ENTRYPOINT ["./myapp"]
```

Build and run:
```bash
docker build -t myapp:latest .
docker run -p 3000:3000 myapp:latest
```

!!! tip "Distroless images"
    For the smallest and most secure runtime, use `gcr.io/distroless/cc-debian12` instead of `debian:bookworm-slim`. It has no shell, no package manager — just your binary.

---

## 2. Environment Configuration

Never hardcode configuration. Read it from environment variables at startup:

```toml
[dependencies]
dotenvy = "0.15"  # loads .env files in development
```

```rust
use std::env;

#[derive(Debug)]
struct Config {
    database_url: String,
    port:         u16,
    jwt_secret:   String,
    log_level:    String,
}

impl Config {
    fn from_env() -> Result<Self, env::VarError> {
        dotenvy::dotenv().ok(); // loads .env if present (dev only)
        Ok(Self {
            database_url: env::var("DATABASE_URL")?,
            port:         env::var("PORT").unwrap_or("3000".into()).parse().unwrap_or(3000),
            jwt_secret:   env::var("JWT_SECRET")?,
            log_level:    env::var("LOG_LEVEL").unwrap_or("info".into()),
        })
    }
}
```

```bash
# .env (never commit to git)
DATABASE_URL=postgresql://user:pass@localhost/mydb
JWT_SECRET=super-secret-key-change-in-prod
PORT=3000
```

---

## 3. Docker Compose for Local Development

```yaml
# docker-compose.yml
version: "3.9"
services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://dev:dev@db:5432/mydb
      - JWT_SECRET=dev-secret
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: mydb
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dev"]
      interval: 5s
      retries: 5
```

```bash
docker compose up --build
```

---

## 4. Kubernetes Deployment

Once your image is pushed to a registry (e.g. Docker Hub, AWS ECR, GCR), deploy it to Kubernetes:

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myregistry/myapp:latest
          ports:
            - containerPort: 3000
          env:
            - name: PORT
              value: "3000"
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: jwt-secret
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            limits:   { cpu: "500m", memory: "128Mi" }
            requests: { cpu: "100m", memory: "64Mi" }
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 3000
  type: LoadBalancer
```

```bash
kubectl apply -f k8s/deployment.yaml
kubectl get pods
kubectl logs -f deployment/myapp
```

---

## 5. Secrets Management

Never put secrets in environment variables baked into images. Use:

**Kubernetes Secrets:**
```bash
kubectl create secret generic myapp-secrets \
  --from-literal=jwt-secret="$(openssl rand -hex 32)"
```

**AWS Secrets Manager / Parameter Store** — read at runtime via SDK:
```toml
[dependencies]
aws-config = "1"
aws-sdk-secretsmanager = "1"
```

```rust
let config = aws_config::load_from_env().await;
let client = aws_sdk_secretsmanager::Client::new(&config);
let secret = client
    .get_secret_value()
    .secret_id("prod/myapp/jwt-secret")
    .send()
    .await?;
```

---

## 6. Serverless with `cargo-lambda` (AWS Lambda)

Rust runs exceptionally well on Lambda — cold starts are milliseconds.

```bash
cargo install cargo-lambda
cargo lambda new my-lambda --http   # creates an HTTP-triggered Lambda
cd my-lambda
cargo lambda watch                  # local development server
cargo lambda build --release
cargo lambda deploy --iam-role arn:aws:iam::123456789:role/lambda-role
```

```rust
use lambda_http::{run, service_fn, Body, Error, Request, Response};

async fn handler(event: Request) -> Result<Response<Body>, Error> {
    let resp = Response::builder()
        .status(200)
        .header("content-type", "text/plain")
        .body("Hello from Rust Lambda!".into())?;
    Ok(resp)
}

#[tokio::main]
async fn main() -> Result<(), Error> {
    run(service_fn(handler)).await
}
```

---

## 7. Health Checks & Readiness

Every cloud-deployed service must expose health endpoints:

```rust
use axum::{Router, routing::get, Json};
use serde::Serialize;

#[derive(Serialize)]
struct Health { status: &'static str, version: &'static str }

async fn health() -> Json<Health> {
    Json(Health { status: "ok", version: env!("CARGO_PKG_VERSION") })
}

let app = Router::new().route("/health", get(health));
```

---

## 8. Continuous Deployment with GitHub Actions

Extend the CI workflow to build and push a Docker image on every merge to `main`:

```yaml
# .github/workflows/deploy-app.yml
- name: Build and push Docker image
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: |
      myregistry/myapp:latest
      myregistry/myapp:${{ github.sha }}

- name: Deploy to Kubernetes
  run: |
    kubectl set image deployment/myapp myapp=myregistry/myapp:${{ github.sha }}
    kubectl rollout status deployment/myapp
```

---

## What's Next?

[Go to Practice :fontawesome-solid-dumbbell:](practice.md){ .md-button .md-button--primary }
[Jump to Challenges :fontawesome-solid-trophy:](challenges.md){ .md-button }
