# Practice Rust for Beginners

> **Learn Rust — From Zero to Production**  
> A structured 6-track programme delivered as a GitHub Pages site.

[![Deploy to GitHub Pages](https://github.com/practice-rust-for-beginners/practice-rust-for-beginners.github.io/actions/workflows/deploy.yml/badge.svg)](https://github.com/practice-rust-for-beginners/practice-rust-for-beginners.github.io/actions/workflows/deploy.yml)

🌐 **Live site:** https://practice-rust-for-beginners.github.io/

---

## Tracks

| # | Track | Focus |
|---|-------|-------|
| 1 | 🦀 Fundamentals | Variables, ownership, traits, error handling |
| 2 | 🌐 Web Development | Axum, Tokio, async, templating |
| 3 | 🔌 APIs | REST, GraphQL, serde, reqwest, OpenAPI |
| 4 | ☁️ Cloud | Docker, Kubernetes, Lambda, CI/CD |
| 5 | 🤖 AI | LLM APIs, Candle, ONNX, embeddings, RAG |
| 6 | 🏢 Enterprise Apps | SQLx, tracing, Kafka, JWT, WASM |

Each track has three sections: **Learn** → **Practice** → **Challenges**.

---

## Local Development

### Prerequisites

- Python 3.9+
- pip

### Setup

```bash
# Clone the repo
git clone https://github.com/practice-rust-for-beginners/practice-rust-for-beginners.github.io
cd practice-rust-for-beginners.github.io

# Create a virtual environment (recommended)
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Start the dev server (live-reload)
mkdocs serve
```

Open http://localhost:8000 in your browser.

### Build static site

```bash
mkdocs build
# Output is in the `site/` directory
```

### Strict build (catches broken links)

```bash
mkdocs build --strict
```

---

## Deployment

The site deploys automatically via GitHub Actions on every push to `main`.

The workflow (`.github/workflows/deploy.yml`):
1. Installs Python + dependencies from `requirements.txt`.
2. Runs `mkdocs gh-deploy --force`.
3. Pushes the built site to the `gh-pages` branch.

GitHub Pages serves the `gh-pages` branch at the live URL above.

---

## Contributing

1. Fork the repository.
2. Create a branch: `git checkout -b track-1-add-closures`.
3. Edit content in `docs/`.
4. Run `mkdocs serve` locally and verify your changes look correct.
5. Run `mkdocs build --strict` — it must pass with zero warnings.
6. Open a pull request.

### Content guidelines

- All code examples must be in ` ```rust ` fences.
- Use `!!! tip`, `!!! warning`, `!!! info`, `??? example` admonitions for callouts.
- Each Learn page ends with navigation buttons to Practice and Challenges.
- Each Practice/Challenges page ends with navigation buttons back and forward.

---

## Tech Stack

| Tool | Purpose |
|------|---------|
| [MkDocs](https://www.mkdocs.org/) | Static site generator |
| [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) | Theme |
| [GitHub Pages](https://pages.github.com/) | Hosting |
| [GitHub Actions](https://github.com/features/actions) | CI/CD |

---

## Licence

Content is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Code examples are licensed under [MIT](https://opensource.org/licenses/MIT).
