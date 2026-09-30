# Track 5 — AI: Learn

Track 5 explores the intersection of Rust and artificial intelligence: calling large language model APIs, running on-device inference, building embedding pipelines, and integrating with vector databases — all from high-performance Rust code.

**Prerequisites:** Tracks 3 & 4 — APIs and Cloud.

---

## 1. Calling LLM APIs — OpenAI

The `async-openai` crate provides a typed Rust client for the OpenAI API.

```toml
[dependencies]
async-openai = "0.23"
tokio        = { version = "1", features = ["full"] }
```

```rust
use async_openai::{Client, types::{
    CreateChatCompletionRequestArgs, ChatCompletionRequestUserMessageArgs,
}};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = Client::new(); // reads OPENAI_API_KEY from env

    let request = CreateChatCompletionRequestArgs::default()
        .model("gpt-4o-mini")
        .messages([
            ChatCompletionRequestUserMessageArgs::default()
                .content("Explain Rust ownership in one paragraph.")
                .build()?
                .into(),
])
        .build()?;

    let response = client.chat().create(request).await?;
    let reply = &response.choices[0].message.content;
    println!("{reply:?}");
    Ok(())
}
```

### Streaming responses

```rust
use async_openai::types::CreateChatCompletionRequestArgs;
use futures::StreamExt;

let mut stream = client.chat().create_stream(request).await?;
while let Some(chunk) = stream.next().await {
    for choice in chunk?.choices {
        if let Some(delta) = choice.delta.content {
            print!("{delta}");
        }
    }
}
```

---

## 2. Calling IBM watsonx.ai

IBM watsonx.ai exposes a REST API. Call it from Rust using `reqwest`:

```toml
[dependencies]
reqwest    = { version = "0.12", features = ["json"] }
serde      = { version = "1",   features = ["derive"] }
serde_json = "1"
```

```rust
use reqwest::Client;
use serde_json::json;

async fn watsonx_generate(
    api_key: &str,
    project_id: &str,
    prompt: &str,
) -> Result<String, reqwest::Error> {
    let client = Client::new();

    // 1. Get IAM access token
    let token_resp: serde_json::Value = client
        .post("https://iam.cloud.ibm.com/identity/token")
        .form(&[("grant_type", "urn:ibm:params:oauth:grant-type:apikey"),
                ("apikey", api_key)])
        .send().await?.json().await?;
    let access_token = token_resp["access_token"].as_str().unwrap_or_default();

    // 2. Call text generation endpoint
    let resp: serde_json::Value = client
        .post("https://us-south.ml.cloud.ibm.com/ml/v1/text/generation?version=2023-05-29")
        .bearer_auth(access_token)
        .json(&json!({
            "model_id": "ibm/granite-13b-chat-v2",
            "input": prompt,
            "parameters": { "max_new_tokens": 200 },
            "project_id": project_id
        }))
        .send().await?.json().await?;

    Ok(resp["results"][0]["generated_text"]
        .as_str().unwrap_or_default().to_string())
}
```

---

## 3. Local Inference with Candle

**Candle** is HuggingFace's pure-Rust ML framework — no Python runtime required.

```toml
[dependencies]
candle-core        = "0.6"
candle-transformers = "0.6"
candle-nn          = "0.6"
hf-hub             = "0.3"
tokenizers         = "0.19"
```

```rust
use candle_core::{Device, Tensor};
use candle_nn::VarBuilder;

// Load a model from HuggingFace Hub
let device = Device::Cpu; // or Device::Cuda(0) for GPU
let api = hf_hub::api::sync::Api::new()?;
let model_dir = api.model("facebook/opt-125m".to_string()).get("model.safetensors")?;
```

!!! info "Candle models"
    Candle ships with ready-to-use implementations of Llama 2/3, Mistral, Falcon, Whisper, BERT, SAM, and more. Check the [candle examples](https://github.com/huggingface/candle/tree/main/candle-examples).

---

## 4. ONNX Runtime — `ort`

ONNX Runtime lets you run any ONNX model (from PyTorch, TensorFlow, scikit-learn) in Rust.

```toml
[dependencies]
ort = "2"
ndarray = "0.16"
```

```rust
use ort::{Environment, Session, SessionBuilder, Value};
use ndarray::Array2;

fn run_model() -> ort::Result<()> {
    let session = Session::builder()?
        .with_model_from_file("model.onnx")?;

    // Build input tensor
    let input = Array2::<f32>::zeros((1, 512));
    let outputs = session.run(ort::inputs!["input" => input.view()]?)?;
    let logits = outputs["logits"].extract_tensor::<f32>()?;
    println!("{:?}", logits.view().shape());
    Ok(())
}
```

---

## 5. Text Embeddings

Embeddings convert text into dense vectors for semantic search and similarity.

```rust
// Using a BERT-family model via Candle
use candle_transformers::models::bert::{BertModel, Config};

fn embed(text: &str, model: &BertModel, tokenizer: &tokenizers::Tokenizer)
    -> candle_core::Result<Vec<f32>>
{
    let encoding = tokenizer.encode(text, true).unwrap();
    let ids = Tensor::new(encoding.get_ids(), &Device::Cpu)?.unsqueeze(0)?;
    let output = model.forward(&ids, &ids.zeros_like()?, None)?;
    // Mean-pool across sequence dimension
    let pooled = output.mean(1)?;
    Ok(pooled.squeeze(0)?.to_vec1::<f32>()?)
}
```

### Cosine similarity

```rust
fn cosine_similarity(a: &[f32], b: &[f32]) -> f32 {
    let dot:   f32 = a.iter().zip(b).map(|(x, y)| x * y).sum();
    let norm_a: f32 = a.iter().map(|x| x * x).sum::<f32>().sqrt();
    let norm_b: f32 = b.iter().map(|x| x * x).sum::<f32>().sqrt();
    dot / (norm_a * norm_b)
}
```

---

## 6. Vector Databases

Once you have embeddings, store and search them efficiently.

### In-process: `usearch`

```toml
[dependencies]
usearch = "2"
```

```rust
use usearch::{Index, IndexOptions, MetricKind, ScalarKind};

let options = IndexOptions {
    dimensions: 384,
    metric: MetricKind::Cos,
    quantization: ScalarKind::F32,
    ..Default::default()
};
let index = Index::new(&options)?;
index.reserve(10_000)?;

index.add(42, &embedding)?;  // key, vector
let results = index.search(&query_embedding, 5)?;  // top-5
```

### Remote: Qdrant via REST

```rust
// POST /collections/docs/points/search
let result: serde_json::Value = reqwest_client
    .post("http://localhost:6333/collections/docs/points/search")
    .json(&json!({
        "vector": query_embedding,
        "limit": 5,
        "with_payload": true
    }))
    .send().await?.json().await?;
```

---

## 7. Building a RAG Pipeline

Retrieval-Augmented Generation (RAG) combines vector search with LLM generation:

```text
User question
    ↓
Embed question (BERT/OpenAI embeddings)
    ↓
Search vector DB for top-k relevant chunks
    ↓
Build prompt: "Context: {chunks}\nQuestion: {question}"
    ↓
Send to LLM (watsonx / OpenAI)
    ↓
Return answer
```

```rust
async fn rag_answer(
    question: &str,
    index: &Index,
    chunks: &[String],
    llm_client: &Client,
) -> String {
    let q_embedding = embed(question);
    let hits = index.search(&q_embedding, 3).unwrap();
    let context: String = hits.keys.iter()
        .map(|&k| chunks[k as usize].clone())
        .collect::<Vec<_>>().join("\n\n");
    let prompt = format!("Context:\n{context}\n\nQuestion: {question}\nAnswer:");
    call_llm(&prompt, llm_client).await
}
```

---

## What's Next?

[Go to Practice](practice.md){ .md-button .md-button--primary }
[Jump to Challenges](challenges.md){ .md-button }
