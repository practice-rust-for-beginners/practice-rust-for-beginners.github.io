# Track 5 — AI: Challenges

---

## Challenge 1 — CLI Rust Tutor

Build an AI-powered Rust tutor CLI:

- Accepts Rust code as input (paste or read from file).
- Sends it to an LLM with a system prompt: "You are an expert Rust teacher. Review the code and suggest improvements. Focus on idiomatic Rust, ownership, and performance."
- Streams the response to the terminal with syntax-highlighted output.
- Optionally: ask follow-up questions in a multi-turn loop.

**Crates:** `async-openai`, `syntect` (for syntax highlighting), `rustyline` (for readline input).

---

## Challenge 2 — Document Q&A System

Build a full RAG (Retrieval-Augmented Generation) pipeline:

1. **Ingest:** Read `.txt` or `.md` files from a directory, chunk them (500-token chunks with 50-token overlap), embed each chunk, store in a `usearch` index.
2. **Query:** Accept a natural language question, embed it, retrieve the top 5 most relevant chunks, build a prompt, call an LLM, return the answer with source citations.
3. **Persist:** Save the index and chunk metadata to disk so it survives restarts.
4. **CLI interface:**
   ```
   rag> ingest ./docs
   rag> ask "What is ownership in Rust?"
   rag> ask "How do I use async/await?"
   ```

---

## Challenge 3 — AI-powered Code Review API

Build an Axum REST API that reviews Rust code:

- `POST /review` — accepts `{"code": "...", "level": "beginner|intermediate|advanced"}` and returns structured feedback as JSON:
  ```json
  {
    "issues": [{"line": 5, "severity": "warning", "message": "..."}],
    "suggestions": ["..."],
    "score": 8.5,
    "summary": "..."
  }
  ```
- Use an LLM with a carefully crafted system prompt that returns valid JSON every time.
- Cache identical code reviews for 1 hour.
- Rate limit to 10 reviews per minute per IP.

---

## Challenge 4 — Real-time Transcription Service

Build a service that transcribes audio using the OpenAI Whisper API:

- `POST /transcribe` — accepts a multipart audio file upload (WAV, MP3, M4A).
- Sends to OpenAI's `whisper-1` model.
- Returns `{"transcript": "...", "language": "en", "duration_seconds": 12.4}`.
- `POST /translate` — same but translates to English.
- Store all transcriptions in memory with timestamps; expose `GET /history`.

---

## Challenge 5 — On-device Text Classifier

Build a fully local (no API calls) text classifier using Candle:

1. Load a quantised BERT model (`bert-base-uncased` or `distilbert`) from HuggingFace Hub at startup.
2. Fine-tune (or use zero-shot) for 3 categories: `positive`, `neutral`, `negative`.
3. Expose as an Axum API: `POST /classify` accepts `{"text": "..."}` and returns `{"label": "positive", "confidence": 0.92}`.
4. Process a batch of 100 texts from a JSON file using `tokio::task::spawn_blocking` for the CPU-bound inference.
5. Measure and report P50/P99 latency.

---

[Back to Practice](practice.md){ .md-button }

[Next Track: Enterprise Apps](../06-enterprise-apps/learn.md){ .md-button .md-button--primary }
