# Project: CLI Chatbot

**Track:** 🤖 AI &nbsp;|&nbsp; **Difficulty:** 🟡 Intermediate &nbsp;|&nbsp; **Crates:** async-openai, tokio, serde

---

## What You'll Build

A streaming terminal chatbot that remembers the full conversation history, supports system prompts, tracks token usage, and lets you reset or switch personas mid-session.

```text
$ cargo run
🦀 Rust Chatbot (model: gpt-4o-mini) — type /help for commands

You: What is the borrow checker?
Assistant: The borrow checker is Rust's compile-time mechanism for
enforcing memory safety rules without a garbage collector...

You: /tokens
Tokens used this session: 342

You: /system You are a pirate who only speaks in pirate dialect.
System prompt set.

You: Tell me about ownership
Assistant: Arrr, ownership be the mightiest rule of the Rust seas!...
```

---

## Tech Stack

```toml
[dependencies]
async-openai = "0.23"
tokio        = { version = "1", features = ["full"] }
serde        = { version = "1", features = ["derive"] }
futures      = "0.3"
```

---

## Step 1 — Session State

```rust
// src/session.rs
use async_openai::types::{
    ChatCompletionRequestMessage,
    ChatCompletionRequestSystemMessageArgs,
    ChatCompletionRequestUserMessageArgs,
    ChatCompletionRequestAssistantMessageArgs,
};

pub struct Session {
    pub messages:     Vec<ChatCompletionRequestMessage>,
    pub system:       Option<String>,
    pub total_tokens: u32,
    pub model:        String,
}

impl Session {
    pub fn new(model: impl Into<String>) -> Self {
        Self { messages: vec![], system: None, total_tokens: 0, model: model.into() }
    }

    pub fn set_system(&mut self, prompt: impl Into<String>) {
        self.system = Some(prompt.into());
        println!("System prompt set.");
    }

    pub fn push_user(&mut self, text: &str) {
        self.messages.push(
            ChatCompletionRequestUserMessageArgs::default()
                .content(text).build().unwrap().into()
        );
    }

    pub fn push_assistant(&mut self, text: &str) {
        self.messages.push(
            ChatCompletionRequestAssistantMessageArgs::default()
                .content(text).build().unwrap().into()
        );
    }

    pub fn build_messages(&self) -> Vec<ChatCompletionRequestMessage> {
        let mut msgs = vec![];
        if let Some(sys) = &self.system {
            msgs.push(
                ChatCompletionRequestSystemMessageArgs::default()
                    .content(sys.clone()).build().unwrap().into()
            );
        }
        msgs.extend_from_slice(&self.messages);
        msgs
    }

    pub fn reset(&mut self) {
        self.messages.clear();
        println!("Conversation cleared.");
    }
}
```

---

## Step 2 — Streaming Response

```rust
// src/chat.rs
use async_openai::{Client, types::CreateChatCompletionRequestArgs};
use futures::StreamExt;
use crate::session::Session;

pub async fn send_and_stream(session: &mut Session) -> String {
    let client = Client::new();

    let request = CreateChatCompletionRequestArgs::default()
        .model(&session.model)
        .messages(session.build_messages())
        .stream(true)
        .build()
        .unwrap();

    let mut stream = client.chat().create_stream(request).await.unwrap();
    let mut full_response = String::new();

    print!("Assistant: ");
    while let Some(Ok(chunk)) = stream.next().await {
        for choice in &chunk.choices {
            if let Some(delta) = &choice.delta.content {
                print!("{delta}");
                full_response.push_str(delta);
                use std::io::Write;
                std::io::stdout().flush().ok();
            }
        }
        // Accumulate token usage if provided
        if let Some(usage) = &chunk.usage {
            session.total_tokens += usage.total_tokens;
        }
    }
    println!(); // newline after streamed response
    full_response
}
```

---

## Step 3 — Command Handling

```rust
// src/commands.rs
use crate::session::Session;

pub enum Action {
    Chat,
    Quit,
    Skip,
}

pub fn handle(input: &str, session: &mut Session) -> Action {
    match input.trim() {
        "/quit" | "/exit" | "/q" => { println!("Goodbye!"); Action::Quit }
        "/reset" | "/clear"      => { session.reset(); Action::Skip }
        "/tokens"                => {
            println!("Tokens used this session: {}", session.total_tokens);
            Action::Skip
        }
        "/model" => {
            println!("Current model: {}", session.model);
            Action::Skip
        }
        "/help" => {
            println!("Commands: /reset /tokens /model /system <prompt> /quit");
            Action::Skip
        }
        _ if input.starts_with("/system ") => {
            session.set_system(&input[8..]);
            Action::Skip
        }
        _ => Action::Chat,
    }
}
```

---

## Step 4 — Main Loop

```rust
// src/main.rs
mod session;
mod chat;
mod commands;

use std::io::{self, Write};
use session::Session;

#[tokio::main]
async fn main() {
    let model = std::env::var("MODEL").unwrap_or("gpt-4o-mini".into());
    println!("🦀 Rust Chatbot (model: {model}) — type /help for commands\n");

    let mut session = Session::new(&model);

    loop {
        print!("You: ");
        io::stdout().flush().unwrap();

        let mut input = String::new();
        if io::stdin().read_line(&mut input).is_err() || input.trim().is_empty() { continue; }
        let input = input.trim().to_string();

        match commands::handle(&input, &mut session) {
            commands::Action::Quit => break,
            commands::Action::Skip => continue,
            commands::Action::Chat => {
                session.push_user(&input);
                let reply = chat::send_and_stream(&mut session).await;
                session.push_assistant(&reply);
            }
        }
    }
}
```

---

## Step 5 — Run It

```bash
export OPENAI_API_KEY=sk-your-key-here
cargo run
```

---

## Stretch Goals

- [ ] Save and load conversation history to/from a JSON file
- [ ] Add `/model gpt-4o` to switch models mid-session
- [ ] Add a `--persona "You are a Rust expert"` CLI flag
- [ ] Add coloured output: user input in blue, assistant in green, system in grey

---

[Back to All Projects](../index.md){ .md-button }


[Next Project: Document Q&A](document-qa.md){ .md-button .md-button--primary }
