# Track 4 — Cloud: Challenges

---

## Challenge 1 — Zero-downtime Rolling Deploy

Build an Axum API and automate a zero-downtime rolling update:

1. The API has `GET /version` returning `{"version": "1.0.0"}`.
2. Write a `Dockerfile` and a `docker-compose.yml` with **2 replicas** behind an nginx load balancer.
3. Update the version to `"2.0.0"` and write a shell script (`deploy.sh`) that:
   - Builds the new image.
   - Replaces one container at a time (update container 1, wait for health check, update container 2).
   - Runs `curl` in a loop every 100ms during the deploy to verify **zero 5xx responses**.

---

## Challenge 2 — Serverless Image Resizer

Build an AWS Lambda function (using `cargo-lambda`) that:

1. Is triggered by `POST /resize` with a multipart image upload (or a JSON `{"url": "..."}` pointing to an image).
2. Resizes the image to 300×300 using the `image` crate.
3. Returns the resized image as base64-encoded PNG in the response body.
4. Test locally with `cargo lambda watch`.

**Crates:** `image = "0.25"`, `base64 = "0.22"`.

---

## Challenge 3 — Multi-environment Config Pipeline

Build a config system that loads from multiple sources in priority order:

1. **Defaults** (hardcoded in code).
2. **Config file** (`config/default.toml` and `config/{env}.toml` where `env` is `development`, `staging`, `production`).
3. **Environment variables** (override anything).

Use the `config` crate. Validate the final config (required fields, valid ranges) and print a redacted summary at startup (mask secrets).

Write unit tests that verify each override layer works correctly.

---

## Challenge 4 — Kubernetes-ready Microservice

Build a production-ready microservice that satisfies all of:

- `GET /health` — liveness probe.
- `GET /ready` — readiness probe (returns 503 until DB connection is established — simulate with a 3-second startup delay).
- Graceful shutdown: on `SIGTERM`, finish in-flight requests, close connections, exit within 30 seconds.
- Structured JSON logs using `tracing` + `tracing-subscriber` with `json` format.
- Prometheus metrics at `GET /metrics` (request count + latency histogram) using `metrics` + `metrics-exporter-prometheus` crates.
- A Kubernetes `Deployment` + `Service` + `HorizontalPodAutoscaler` (scale 1–5 on CPU > 60%).

---

## Challenge 5 — CI/CD Pipeline

Write a complete GitHub Actions pipeline for a Rust web service:

1. **CI** (on every PR): `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo test`, `cargo build --release`, Docker image build (no push).
2. **Deploy to staging** (on push to `main`): build + push image to registry, deploy to staging namespace.
3. **Deploy to production** (on git tag `v*`): deploy tagged image to production namespace, create a GitHub Release with changelog.
4. Use reusable workflows and secrets for registry credentials.

The pipeline must cache `~/.cargo/registry` and `target/` between runs.

---

[Back to Practice](practice.md){ .md-button }

[Next Track: AI](../05-ai/learn.md){ .md-button .md-button--primary }
