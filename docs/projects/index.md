# All Projects

Build real Rust applications from scratch. Every project includes a full step-by-step guide, code snippets, and a recommended tech stack. Projects are organised by track — work through them in order, or jump to any that interests you.

!!! tip "Prerequisites"
    Complete the **Learn** section of the corresponding track before starting its projects. Each project assumes you are comfortable with the concepts covered there.

---

## Difficulty levels

| Badge | Meaning |
|-------|---------|
| 🟢 Beginner | Uses only Track 1–2 concepts. Ideal for your first Rust project. |
| 🟡 Intermediate | Requires Track 3–4 concepts: async, APIs, serde, Docker. |
| 🔴 Advanced | Full-stack, multi-crate, or production-grade. Requires Track 5–6. |

---

## 🦀 Fundamentals Projects

<div class="grid cards" markdown>

-   **CLI Calculator** — 🟢 Beginner

    ---

    A command-line calculator with operator parsing, error handling, and history. Pure Rust standard library.

    [:octicons-arrow-right-24: Build it](01-fundamentals/cli-calculator.md)

-   **Text Adventure Game** — 🟢 Beginner

    ---

    A terminal-based text adventure with rooms, items, and NPCs. Practice enums, pattern matching, and structs.

    [:octicons-arrow-right-24: Build it](01-fundamentals/text-adventure.md)

-   **File Organiser** — 🟢 Beginner

    ---

    A CLI tool that scans a directory and sorts files into subfolders by extension. Uses `std::fs` and iterators.

    [:octicons-arrow-right-24: Build it](01-fundamentals/file-organiser.md)

</div>

---

## 🌐 Web Development Projects

<div class="grid cards" markdown>

-   **Personal Blog Engine** — 🟡 Intermediate

    ---

    A server-side rendered blog with posts, tags, and a Markdown renderer. Axum + Askama templates.

    [:octicons-arrow-right-24: Build it](02-web-development/blog-engine.md)

-   **Link Shortener** — 🟡 Intermediate

    ---

    A URL shortener service with redirect tracking and click analytics. Axum + in-memory state.

    [:octicons-arrow-right-24: Build it](02-web-development/link-shortener.md)

-   **Real-time Chat App** — 🟡 Intermediate

    ---

    A multi-room chat server using WebSockets and broadcast channels. Tokio + Axum WS.

    [:octicons-arrow-right-24: Build it](02-web-development/realtime-chat.md)

</div>

---

## 🔌 API Projects

<div class="grid cards" markdown>

-   **Recipe API** — 🟡 Intermediate

    ---

    A fully documented REST API for recipes with search, filters, and OpenAPI spec. Axum + serde + utoipa.

    [:octicons-arrow-right-24: Build it](03-apis/recipe-api.md)

-   **GitHub Dashboard** — 🟡 Intermediate

    ---

    A CLI dashboard that aggregates your GitHub stats, PRs, and issues using the GitHub REST API.

    [:octicons-arrow-right-24: Build it](03-apis/github-dashboard.md)

-   **Notification Service** — 🟡 Intermediate

    ---

    A webhook-driven notification service that fans out events to email, Slack, and webhooks.

    [:octicons-arrow-right-24: Build it](03-apis/notification-service.md)

</div>

---

## ☁️ Cloud Projects

<div class="grid cards" markdown>

-   **Containerised Microservice** — 🟡 Intermediate

    ---

    Package an API into a minimal Docker image and deploy it with a docker-compose stack including Postgres.

    [:octicons-arrow-right-24: Build it](04-cloud/containerised-microservice.md)

-   **Serverless URL Shortener** — 🟡 Intermediate

    ---

    Rebuild the link shortener as an AWS Lambda function using `cargo-lambda`. DynamoDB for persistence.

    [:octicons-arrow-right-24: Build it](04-cloud/serverless-url-shortener.md)

-   **CI/CD Pipeline** — 🔴 Advanced

    ---

    A full GitHub Actions pipeline: lint → test → build → containerise → deploy to Kubernetes on tag.

    [:octicons-arrow-right-24: Build it](04-cloud/cicd-pipeline.md)

</div>

---

## 🤖 AI Projects

<div class="grid cards" markdown>

-   **CLI Chatbot** — 🟡 Intermediate

    ---

    A streaming terminal chatbot with conversation history, system prompts, and token counting.

    [:octicons-arrow-right-24: Build it](05-ai/cli-chatbot.md)

-   **Document Q&A** — 🔴 Advanced

    ---

    A RAG pipeline: ingest documents, embed with BERT, store in a vector index, answer questions with an LLM.

    [:octicons-arrow-right-24: Build it](05-ai/document-qa.md)

-   **Code Review Bot** — 🔴 Advanced

    ---

    An Axum API that runs `cargo clippy` and sends code to an LLM for review. Returns structured JSON feedback.

    [:octicons-arrow-right-24: Build it](05-ai/code-review-bot.md)

</div>

---

## 🏢 Enterprise App Projects

<div class="grid cards" markdown>

-   **E-commerce API** — 🔴 Advanced

    ---

    A production-ready e-commerce backend: auth, products, cart, orders, transactions. SQLx + migrations + JWT.

    [:octicons-arrow-right-24: Build it](06-enterprise-apps/ecommerce-api.md)

-   **Event-driven Analytics** — 🔴 Advanced

    ---

    A high-throughput event ingestion service with real-time aggregation and Prometheus metrics.

    [:octicons-arrow-right-24: Build it](06-enterprise-apps/event-driven-analytics.md)

-   **Microservice Mesh** — 🔴 Advanced

    ---

    Three Rust microservices communicating via NATS, each with its own DB, behind an API gateway.

    [:octicons-arrow-right-24: Build it](06-enterprise-apps/microservice-mesh.md)

</div>
