# Project: Document Q&A

**Track:** 🤖 AI &nbsp;|&nbsp; **Difficulty:** 🔴 Advanced &nbsp;|&nbsp; **Crates:** async-openai, usearch, tokio, serde

---

## What You'll Build

A RAG (Retrieval-Augmented Generation) pipeline that ingests local documents, builds a vector index from their embeddings, and answers natural language questions by finding relevant chunks and passing them to an LLM.

```text
$ cargo run -- ingest ./docs
Chunking 12 documents...
Embedding 87 chunks... done.
Index saved to ./data/index.bin

$ cargo run -- ask "How does Rust handle memory safety?"
Searching 87 chunks...

Sources:
  [1] docs/ownership.md (score: 0.91)
  [2] docs/borrowing.md (score: 0.87)

Answer:
Rust handles memory safety through its ownership system, which enforces
three rules at compile time: each value has exactly one owner...
```

---

## Tech Stack

```toml
[dependencies]
async-openai = "0.23"
usearch      = "2"
tokio        = { version = "1", features = ["full"] }
serde        = { version = "1", features = ["derive"] }
serde_json   = "1"
walkdir      = "2"
```

---

## Step 1 — Chunking

```rust
// src/chunker.rs
pub struct Chunk {
    pub source: String,
    pub text:   String,
    pub id:     u64,
}

const CHUNK_SIZE: usize    = 500;
const CHUNK_OVERLAP: usize = 50;

pub fn chunk_text(text: &str, source: &str, start_id: u64) -> Vec<Chunk> {
    let words: Vec<&str> = text.split_whitespace().collect();
    let mut chunks = Vec::new();
    let mut i = 0usize;
    let mut id = start_id;

    while i < words.len() {
        let end = (i + CHUNK_SIZE).min(words.len());
        let chunk_text = words[i..end].join(" ");
        chunks.push(Chunk { source: source.to_string(), text: chunk_text, id });
        id += 1;
        if end == words.len() { break; }
        i += CHUNK_SIZE - CHUNK_OVERLAP;
    }
    chunks
}
```

---

## Step 2 — Embedding

```rust
// src/embedder.rs
use async_openai::{Client, types::CreateEmbeddingRequestArgs};

pub async fn embed_batch(texts: &[String]) -> Vec<Vec<f32>> {
    let client = Client::new();
    let request = CreateEmbeddingRequestArgs::default()
        .model("text-embedding-3-small")
        .input(texts.to_vec())
        .build().unwrap();

    let response = client.embeddings().create(request).await.unwrap();
    let mut result = vec![vec![]; texts.len()];
    for item in response.data {
        result[item.index as usize] = item.embedding;
    }
    result
}
```

---

## Step 3 — Vector Index

```rust
// src/index_store.rs
use usearch::{Index, IndexOptions, MetricKind, ScalarKind};
use serde::{Deserialize, Serialize};
use std::{fs, path::Path};

#[derive(Serialize, Deserialize)]
pub struct Metadata {
    pub chunks: Vec<ChunkMeta>,
}

#[derive(Serialize, Deserialize, Clone)]
pub struct ChunkMeta {
    pub id:     u64,
    pub source: String,
    pub text:   String,
}

pub fn new_index(dimensions: usize) -> Index {
    Index::new(&IndexOptions {
        dimensions,
        metric:       MetricKind::Cos,
        quantization: ScalarKind::F32,
        ..Default::default()
    }).unwrap()
}

pub fn save(index: &Index, meta: &Metadata, dir: &str) {
    fs::create_dir_all(dir).unwrap();
    index.save(format!("{dir}/index.bin")).unwrap();
    let json = serde_json::to_string(meta).unwrap();
    fs::write(format!("{dir}/meta.json"), json).unwrap();
}

pub fn load(dir: &str, dimensions: usize) -> (Index, Metadata) {
    let index = new_index(dimensions);
    index.load(format!("{dir}/index.bin")).unwrap();
    let json = fs::read_to_string(format!("{dir}/meta.json")).unwrap();
    (index, serde_json::from_str(&json).unwrap())
}
```

---

## Step 4 — Ingest Command

```rust
// src/ingest.rs
use walkdir::WalkDir;
use std::fs;
use crate::{chunker::chunk_text, embedder::embed_batch, index_store::{new_index, save, Metadata, ChunkMeta}};

pub async fn run(docs_dir: &str, data_dir: &str) {
    let mut all_chunks = Vec::new();
    let mut id = 0u64;

    for entry in WalkDir::new(docs_dir).into_iter().flatten() {
        let path = entry.path();
        if path.is_file() && matches!(path.extension().and_then(|e| e.to_str()), Some("md" | "txt")) {
            let text = fs::read_to_string(path).unwrap_or_default();
            let source = path.to_string_lossy().to_string();
            let chunks = chunk_text(&text, &source, id);
            id += chunks.len() as u64;
            all_chunks.extend(chunks);
        }
    }

    println!("Chunking {} documents → {} chunks", docs_dir, all_chunks.len());

    let texts: Vec<String> = all_chunks.iter().map(|c| c.text.clone()).collect();
    let embeddings = embed_batch(&texts).await;
    let dims = embeddings[0].len();

    let index = new_index(dims);
    index.reserve(all_chunks.len()).unwrap();

    for (chunk, emb) in all_chunks.iter().zip(&embeddings) {
        index.add(chunk.id, emb).unwrap();
    }

    let meta = Metadata {
        chunks: all_chunks.iter().map(|c| ChunkMeta {
            id: c.id, source: c.source.clone(), text: c.text.clone()
        }).collect()
    };

    save(&index, &meta, data_dir);
    println!("Index saved to {data_dir}/");
}
```

---

## Step 5 — Ask Command

```rust
// src/ask.rs
use async_openai::{Client, types::{CreateChatCompletionRequestArgs, ChatCompletionRequestUserMessageArgs}};
use crate::{embedder::embed_batch, index_store::load};

pub async fn run(question: &str, data_dir: &str) {
    let (index, meta) = load(data_dir, 1536);

    let q_emb = embed_batch(&[question.to_string()]).await;
    let results = index.search(&q_emb[0], 3).unwrap();

    println!("\nSources:");
    let mut context = String::new();
    for (i, &key) in results.keys.iter().enumerate() {
        if let Some(chunk) = meta.chunks.iter().find(|c| c.id == key) {
            println!("  [{}] {} (score: {:.2})", i+1, chunk.source, results.distances[i]);
            context.push_str(&format!("[{}] {}\n\n", i+1, chunk.text));
        }
    }

    let prompt = format!("Answer the question using only the context below.\n\nContext:\n{context}\nQuestion: {question}");
    let client = Client::new();
    let request = CreateChatCompletionRequestArgs::default()
        .model("gpt-4o-mini")
        .messages([ChatCompletionRequestUserMessageArgs::default()
            .content(prompt).build().unwrap().into()])
        .build().unwrap();

    let response = client.chat().create(request).await.unwrap();
    let answer = &response.choices[0].message.content;
    println!("\nAnswer:\n{}", answer.as_deref().unwrap_or("No answer"));
}
```

---

## Step 6 — Main

```rust
// src/main.rs
mod chunker;
mod embedder;
mod index_store;
mod ingest;
mod ask;

#[tokio::main]
async fn main() {
    let args: Vec<String> = std::env::args().collect();
    match args.get(1).map(String::as_str) {
        Some("ingest") => ingest::run(args.get(2).unwrap(), "./data").await,
        Some("ask")    => ask::run(args.get(2).unwrap(), "./data").await,
        _ => eprintln!("Usage: rag ingest <dir> | rag ask <question>"),
    }
}
```

---

## Stretch Goals

- [ ] Add `update` command to re-embed only changed files
- [ ] Add multi-turn follow-up questions that include previous Q&A in context
- [ ] Build an Axum HTTP API wrapper around the ask logic
- [ ] Replace OpenAI embeddings with a local Candle BERT model

---

[Back to All Projects](../index.md){ .md-button }


[Next Project: Code Review Bot](code-review-bot.md){ .md-button .md-button--primary }
