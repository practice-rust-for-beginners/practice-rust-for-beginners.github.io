# Track 3 — APIs: Challenges

---

## Challenge 1 — Weather Dashboard API

Build a REST API that aggregates data from the [Open-Meteo](https://open-meteo.com/) free weather API (no key required):

- `GET /weather/:city` — return current temperature, wind speed, and weather code for a city.
  - Use Open-Meteo's geocoding endpoint to resolve city → lat/lon.
  - Then call the forecast endpoint for current conditions.
- `GET /weather/:city/forecast` — return the next 7-day daily high/low temperature forecast.
- Cache results for 10 minutes in `Arc<RwLock<HashMap<String, CachedResponse>>>` (store timestamp alongside data).

---

## Challenge 2 — Webhook Relay

Build a webhook receiver and relay service:

- `POST /webhook/receive` — accepts any JSON payload, stores it in a queue.
- `POST /webhook/register` — accepts `{"url": "https://..."}` to register a target URL.
- A background `tokio::task` drains the queue and forwards each event to all registered URLs via `reqwest`.
- `GET /webhook/events` — returns the last 50 received events with timestamps.
- `GET /webhook/deliveries` — returns delivery status (success/fail + HTTP status) for each forwarded event.

---

## Challenge 3 — Full Auth System

Build a complete authentication API:

- `POST /auth/register` — create a user (hash password with `argon2` or `bcrypt` crate).
- `POST /auth/login` — verify credentials, issue JWT access token (15 min) + refresh token (7 days).
- `POST /auth/refresh` — exchange a valid refresh token for a new access token.
- `POST /auth/logout` — invalidate the refresh token.
- `GET /me` — protected endpoint returning the current user's profile.

Store users and refresh tokens in memory (`HashMap`). Model roles: `admin` and `user`.

---

## Challenge 4 — REST + GraphQL Dual API

Build a single service that exposes the **same data** through both a REST API and a GraphQL API:

- Data model: `Article { id, title, content, author, tags, created_at }`.
- REST: full CRUD under `/api/articles`.
- GraphQL: queries (`articles`, `article(id)`, `articlesByTag(tag)`) and mutations (`createArticle`, `deleteArticle`).
- Both APIs share the same in-memory `Arc<RwLock<HashMap<u32, Article>>>` state.
- Serve Swagger UI at `/swagger-ui` and GraphiQL at `/graphiql`.

---

## Challenge 5 — API Gateway

Build a simple API gateway in front of two downstream services (you can run them locally):

- The gateway runs on `:3000`.
- Routes `GET /users/*` to `http://localhost:3001`.
- Routes `GET /products/*` to `http://localhost:3002`.
- Adds a `X-Request-Id` header (UUID) to every proxied request.
- Logs each request: method, path, upstream target, response status, latency in ms.
- Returns `503 Service Unavailable` if the upstream times out or is unreachable.

**Crates:** `reqwest`, `uuid`, `tracing`.

---

[Back to Practice](practice.md){ .md-button }
[Next Track: Cloud](../04-cloud/learn.md){ .md-button .md-button--primary }
