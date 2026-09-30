# Track 5 — AI: Practice

---

## Exercise 1 — OpenAI Chat Completion

Write a Rust program that:
1. Reads a question from stdin.
2. Sends it to `gpt-4o-mini` using `async-openai`.
3. Prints the response.
4. Loops until the user types `"exit"`.

Set `OPENAI_API_KEY` from your environment.

---

## Exercise 2 — watsonx.ai Text Generation

Call the IBM watsonx.ai REST API to generate text:

1. Authenticate with your IBM Cloud API key to get an IAM token.
2. Send a prompt to `ibm/granite-13b-chat-v2`.
3. Print the generated text.
4. Wrap the API call in a reusable `WatsonxClient` struct with methods:
   - `new(api_key, project_id) -> Self`
   - `generate(prompt, max_tokens) -> Result<String, Error>`

---

## Exercise 3 — Batch Embedding Generator

Write a program that:
1. Reads a file `documents.txt` where each line is a document.
2. Calls the OpenAI embeddings API (`text-embedding-3-small`) for each line.
3. Saves the embeddings to a `embeddings.json` file: `[{"text": "...", "vector": [...]}]`.
4. Reports total tokens used.

Use `tokio::task::spawn` to send batches of 10 requests concurrently.

---

## Exercise 4 — Semantic Similarity

Using the embeddings from Exercise 3:
1. Load `embeddings.json`.
2. Accept a query from stdin.
3. Embed the query.
4. Print the top 3 most similar documents with their cosine similarity scores.

---

## Exercise 5 — CLI Chatbot

Build a terminal chatbot that:
1. Maintains conversation history (all previous messages).
2. Streams the response token by token (use `create_stream`).
3. Supports `/reset` to clear history.
4. Supports `/system <message>` to set a system prompt.
5. Counts and displays total tokens used per session.

---

## Exercise 6 — ONNX Inference

1. Download a pre-quantized ONNX model from HuggingFace (e.g. `optimum/bert-base-uncased` for sequence classification).
2. Write a Rust program using `ort` that:
   - Loads the model.
   - Tokenises input text.
   - Runs inference.
   - Prints the top predicted class and confidence score.

---

[Go to Challenges](challenges.md){ .md-button .md-button--primary }
[Back to Learn](learn.md){ .md-button }
