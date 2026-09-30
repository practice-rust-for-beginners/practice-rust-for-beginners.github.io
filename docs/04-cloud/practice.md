# Track 4 — Cloud: Practice

---

## Exercise 1 — Dockerfile for Your API

Take the books API from Track 3 and containerise it:

1. Write a multi-stage `Dockerfile` (builder + runtime stages).
2. The runtime image must be based on `debian:bookworm-slim` or smaller.
3. The final image must be under **50 MB** (use `docker images` to check).
4. Run `docker run -p 3000:3000 books-api` and verify all endpoints still work.

---

## Exercise 2 — Config from Environment

Refactor the books API so all configuration comes from environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `3000` | Port to listen on |
| `LOG_LEVEL` | `info` | Tracing level |
| `API_KEY` | — (required) | API key for auth middleware |

If `API_KEY` is not set, the app must print an error and exit with code 1.

Test with:
```bash
API_KEY=mysecret PORT=8080 cargo run
```

---

## Exercise 3 — Docker Compose Stack

Build a `docker-compose.yml` that runs:
- The books API on port `3000`.
- A Redis container (`redis:7-alpine`) on port `6379`.
- Connect the API to Redis via the `REDIS_URL` env var (for future caching — just read and log the var for now).

Run `docker compose up --build` and confirm both containers start and the API responds.

---

## Exercise 4 — Health & Readiness Endpoints

Add two endpoints to any of your previous APIs:

- `GET /health` — returns `{"status": "ok", "version": "1.0.0"}` (always 200).
- `GET /ready` — returns `{"ready": true}` if the app is ready to serve, `{"ready": false, "reason": "..."}` with status 503 if not.

Simulate unreadiness by adding a `--delay` flag that makes the app wait 10 seconds before marking itself ready.

---

## Exercise 5 — AWS Lambda Function

Create a Rust Lambda function using `cargo-lambda`:

1. `cargo lambda new greeting --http`
2. Handler reads `name` from the query string: `GET /greet?name=Ferris`.
3. Returns `{"message": "Hello, Ferris!"}`.
4. Test locally with `cargo lambda watch` + `curl`.
5. Build with `cargo lambda build --release`.

---

## Exercise 6 — Kubernetes Manifests

Write Kubernetes YAML for the books API (no live cluster needed — just write valid YAML):

- `Deployment` with 2 replicas, resource limits, and a readiness probe on `/health`.
- `Service` of type `ClusterIP`.
- `Secret` named `books-api-secrets` with an `api-key` key.
- `ConfigMap` with `PORT=3000` and `LOG_LEVEL=info`.

Validate the YAML with `kubectl apply --dry-run=client -f k8s/`.

---

[Go to Challenges](challenges.md){ .md-button .md-button--primary }

[Back to Learn](learn.md){ .md-button }
