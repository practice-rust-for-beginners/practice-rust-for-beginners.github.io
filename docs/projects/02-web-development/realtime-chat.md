# Project: Real-time Chat App

**Track:** 🌐 Web Development &nbsp;|&nbsp; **Difficulty:** 🟡 Intermediate &nbsp;|&nbsp; **Crates:** axum, tokio, serde

---

## What You'll Build

A multi-room WebSocket chat server. Users pick a username, join a room, and see messages from everyone in that room in real time. The browser client is a single self-contained HTML file served by the same Axum server.

```text
ws://localhost:3000/ws?room=rust&user=ferris

{"type":"join",    "user":"ferris",  "room":"rust", "text":"ferris joined"}
{"type":"message", "user":"ferris",  "room":"rust", "text":"Hello!"}
{"type":"leave",   "user":"ferris",  "room":"rust", "text":"ferris left"}
```

---

## Tech Stack

| Component | Crate |
|-----------|-------|
| HTTP + WebSocket | `axum 0.7` (ws feature) |
| Async runtime | `tokio 1` |
| Broadcast | `tokio::sync::broadcast` |
| Serialisation | `serde_json` |
| Static HTML | `tower-http ServeDir` |

```toml
[dependencies]
axum       = { version = "0.7", features = ["ws", "json"] }
tokio      = { version = "1",   features = ["full"] }
serde      = { version = "1",   features = ["derive"] }
serde_json = "1"
tower-http = { version = "0.5", features = ["fs"] }
```

---

## Step 1 — Message Types

```rust
// src/message.rs
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "lowercase")]
pub enum ChatMessage {
    Join    { user: String, room: String, text: String },
    Message { user: String, room: String, text: String },
    Leave   { user: String, room: String, text: String },
    Error   { text: String },
}
```

---

## Step 2 — App State with Per-room Broadcasts

```rust
// src/state.rs
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use tokio::sync::broadcast;
use crate::message::ChatMessage;

const CHANNEL_CAPACITY: usize = 128;

#[derive(Clone)]
pub struct AppState {
    rooms: Arc<Mutex<HashMap<String, broadcast::Sender<ChatMessage>>>>,
}

impl AppState {
    pub fn new() -> Self {
        Self { rooms: Arc::new(Mutex::new(HashMap::new())) }
    }

    pub fn get_or_create_room(&self, room: &str) -> broadcast::Sender<ChatMessage> {
        let mut rooms = self.rooms.lock().unwrap();
        rooms.entry(room.to_string())
            .or_insert_with(|| broadcast::channel(CHANNEL_CAPACITY).0)
            .clone()
    }
}
```

---

## Step 3 — WebSocket Handler

```rust
// src/ws.rs
use axum::{
    extract::{Query, State, WebSocketUpgrade},
    response::IntoResponse,
};
use axum::extract::ws::{Message, WebSocket};
use serde::Deserialize;
use futures::{sink::SinkExt, stream::StreamExt};
use crate::{message::ChatMessage, state::AppState};

#[derive(Deserialize)]
pub struct WsParams { room: String, user: String }

pub async fn handler(
    ws: WebSocketUpgrade,
    Query(params): Query<WsParams>,
    State(state): State<AppState>,
) -> impl IntoResponse {
    ws.on_upgrade(move |socket| handle_socket(socket, params.room, params.user, state))
}

async fn handle_socket(socket: WebSocket, room: String, user: String, state: AppState) {
    let sender = state.get_or_create_room(&room);
    let mut receiver = sender.subscribe();
    let (mut ws_tx, mut ws_rx) = socket.split();

    // Announce join
    let _ = sender.send(ChatMessage::Join {
        user: user.clone(), room: room.clone(),
        text: format!("{user} joined #{room}"),
    });

    // Task: forward broadcast messages → this websocket client
    let mut send_task = tokio::spawn(async move {
        while let Ok(msg) = receiver.recv().await {
            let json = serde_json::to_string(&msg).unwrap();
            if ws_tx.send(Message::Text(json)).await.is_err() { break; }
        }
    });

    // Task: forward incoming WS messages → broadcast channel
    let sender2 = state.get_or_create_room(&room);
    let user2 = user.clone();
    let room2 = room.clone();
    let mut recv_task = tokio::spawn(async move {
        while let Some(Ok(Message::Text(text))) = ws_rx.next().await {
            let msg = ChatMessage::Message {
                user: user2.clone(), room: room2.clone(), text,
            };
            let _ = sender2.send(msg);
        }
    });

    // Wait for either task to finish (client disconnect)
    tokio::select! {
        _ = &mut send_task => recv_task.abort(),
        _ = &mut recv_task => send_task.abort(),
    }

    // Announce leave
    let _ = state.get_or_create_room(&room).send(ChatMessage::Leave {
        user: user.clone(), room: room.clone(),
        text: format!("{user} left #{room}"),
    });
}
```

---

## Step 4 — Serve a Browser Client

```rust
// src/main.rs
mod message;
mod state;
mod ws;

use axum::{Router, routing::get};
use tower_http::services::ServeDir;

#[tokio::main]
async fn main() {
    let state = state::AppState::new();

    let app = Router::new()
        .route("/ws", get(ws::handler))
        .nest_service("/", ServeDir::new("static"))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("Chat server at http://localhost:3000");
    axum::serve(listener, app).await.unwrap();
}
```

Create `static/index.html`:

```html
<!DOCTYPE html>
<html>
<head><title>Rust Chat</title>
<style>
  body { font-family: system-ui; max-width: 600px; margin: 2rem auto; }
  #messages { border: 1px solid #ccc; height: 300px; overflow-y: auto; padding: 8px; margin-bottom: 8px; }
  .join  { color: green; } .leave { color: red; } .msg { color: #333; }
  input, button { padding: 6px; }
</style>
</head>
<body>
<h2>Rust Chat</h2>
<div>Room: <input id="room" value="rust">  User: <input id="user" value="guest">
  <button onclick="connect()">Connect</button></div>
<div id="messages"></div>
<input id="text" placeholder="Message..." style="width:80%">
<button onclick="send()">Send</button>
<script>
let ws;
function connect() {
  const room = document.getElementById('room').value;
  const user = document.getElementById('user').value;
  ws = new WebSocket(`ws://localhost:3000/ws?room=${room}&user=${user}`);
  ws.onmessage = e => {
    const m = JSON.parse(e.data);
    const div = document.getElementById('messages');
    const p = document.createElement('p');
    p.className = m.type;
    p.textContent = m.type === 'message' ? `${m.user}: ${m.text}` : m.text;
    div.appendChild(p);
    div.scrollTop = div.scrollHeight;
  };
}
function send() {
  const t = document.getElementById('text');
  if (ws && t.value) { ws.send(t.value); t.value = ''; }
}
document.getElementById('text').addEventListener('keydown', e => { if (e.key === 'Enter') send(); });
</script>
</body>
</html>
```

---

## Step 5 — Run It

```bash
mkdir static
# (create static/index.html from above)
cargo run
# Open http://localhost:3000 in two browser tabs
```

---

## Stretch Goals

- [ ] Add a `/rooms` endpoint listing active rooms and user counts
- [ ] Add private messaging: `@username your message`
- [ ] Persist the last 50 messages per room and send them to new joiners
- [ ] Add rate limiting: max 5 messages per second per user

---

[Back to All Projects](../index.md){ .md-button }


[Next Track: API Projects](../03-apis/recipe-api.md){ .md-button .md-button--primary }
