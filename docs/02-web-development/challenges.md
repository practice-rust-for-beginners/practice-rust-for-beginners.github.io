# Track 2 — Web Development: Challenges

Five open-ended challenges that require you to combine async Rust, Axum, routing, state management, and templating.

---

## Challenge 1 — URL Shortener Service

Build a URL shortener HTTP service:

- `POST /shorten` — accepts JSON `{"url": "https://..."}`, returns a short code (e.g. `{"short": "http://localhost:3000/r/aB3xY"}`).
- `GET /r/:code` — redirects (HTTP 302) to the original URL, or 404 if not found.
- `GET /stats/:code` — returns JSON with the original URL and hit count.

Store all data in `Arc<RwLock<HashMap<String, ShortEntry>>>`. Generate short codes using random 5-character alphanumeric strings.

**Crates to add:** `rand = "0.8"` for code generation.

---

## Challenge 2 — Chat Room with WebSockets

Build a multi-user chat server:

- `GET /ws` — WebSocket endpoint.
- When a client connects, ask for a username (first message).
- Broadcast every subsequent message to **all** connected clients in format `"[alice]: hello"`.
- When a client disconnects, broadcast `"[alice] left the chat"`.

**Hint:** Use `tokio::sync::broadcast` for fan-out. Store `Arc<broadcast::Sender<String>>` in `AppState`.

---

## Challenge 3 — Todo App with Server-Side HTML

Build a fully functional todo app with server-side rendered HTML (no JavaScript framework):

- `GET /` — lists all todos with checkboxes (done/undone toggle via POST).
- `POST /todos` — create a new todo (form submit).
- `POST /todos/:id/toggle` — mark done/undone.
- `POST /todos/:id/delete` — delete.

Use HTML `<form>` elements with Askama templates. Store todos in `Arc<Mutex<Vec<Todo>>>`.

The app must look decent — include basic CSS inline in the template.

---

## Challenge 4 — Rate-limited API

Build an API server with rate limiting:

- Any route under `/api/` is rate-limited to **10 requests per 60 seconds per IP address**.
- If the limit is exceeded, return `429 Too Many Requests` with a `Retry-After` header.
- Implement the rate limiting as a **Tower middleware** using `Arc<Mutex<HashMap<IpAddr, RateBucket>>>`.
- `GET /api/time` returns the current UTC time as JSON.
- `GET /api/echo?msg=hello` returns `{"echo": "hello"}`.

---

## Challenge 5 — Static Site Server with Directory Listing

Build a file server that:

1. Serves files from a `./public` directory.
2. For directories, renders an HTML page listing files and subdirectories with links (like nginx's autoindex).
3. Displays file sizes and last-modified timestamps.
4. Shows a 404 page (styled HTML, not plain text) for missing files.

Use `std::fs` for directory scanning and Askama for the HTML listing template.

---

[Back to Practice :fontawesome-solid-dumbbell:](practice.md){ .md-button }
[Next Track: APIs :fontawesome-solid-arrow-right:](../03-apis/learn.md){ .md-button .md-button--primary }
