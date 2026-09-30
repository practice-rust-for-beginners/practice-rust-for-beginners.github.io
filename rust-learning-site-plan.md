# Rust Learning Site — Build Plan

## Top-Level Overview

**Goal:** Build a GitHub Pages site using **MkDocs + Material theme** for
[practice-rust-for-beginners.github.io](https://github.com/practice-rust-for-beginners/practice-rust-for-beginners.github.io)
that delivers structured Rust education across 6 progressive tracks.

**Tagline:** _"Learn Rust — From Zero to Production"_

**Approach:**
- Scaffold MkDocs project with the Material theme configured to a Rust-themed orange/dark palette.
- Organise content into 6 numbered track directories under `docs/`, each with three sub-pages: `learn.md`, `practice.md`, `challenges.md`.
- Wire up a GitHub Actions workflow that builds and deploys to `gh-pages` on every push to `main`.
- Pin all Python dependencies in `requirements.txt` for reproducible builds.

**Out of scope (this plan):** Actual educational prose content — that is authored in a separate phase after the skeleton is confirmed.

---

## Sub-Task 1 — Repository Skeleton & Toolchain Config

**Status:** `[ ] pending`

### Intent
Create all configuration and infrastructure files so the site can be built and deployed immediately, even before content is written.

### Expected Outcomes
- `mkdocs.yml` at repo root configures site name, theme, navigation, and Material options.
- `requirements.txt` at repo root pins `mkdocs`, `mkdocs-material`, and any required plugins.
- `.github/workflows/deploy.yml` auto-builds and deploys to the `gh-pages` branch on every push to `main`.
- A minimal placeholder `docs/index.md` home page so the build does not fail.
- The site is reachable at `https://practice-rust-for-beginners.github.io/` after the first push.

### Todo List
1. Create `requirements.txt` pinning:
   - `mkdocs>=1.6`
   - `mkdocs-material>=9.5`
   - `mkdocs-minify-plugin>=0.8`
2. Create `mkdocs.yml` with:
   - `site_name`, `site_url`, `site_description`, `site_author`
   - `theme: material` block with Rust orange primary + dark mode defaults
   - `nav:` skeleton referencing all track pages (stubs are fine)
   - `plugins:` section enabling `search` and `minify`
   - `extra:` block for social links and the Rust gear logo
3. Create `.github/workflows/deploy.yml` with:
   - Trigger: `push` to `main`
   - Steps: checkout → setup Python → pip install → `mkdocs gh-deploy --force`
4. Create `docs/index.md` as the landing page (placeholder content).
5. Create `docs/overrides/` directory placeholder for future theme customisation.

### Relevant Context
- MkDocs docs: https://www.mkdocs.org/user-guide/configuration/
- Material theme: https://squidfunk.github.io/mkdocs-material/
- GitHub Pages deploy action pattern: `mkdocs gh-deploy --force`

---

## Sub-Task 2 — Content Directory Structure

**Status:** `[ ] pending`

### Intent
Create all 6 track directories under `docs/` and stub out every `learn.md`, `practice.md`, and `challenges.md` file so the full nav is populated and buildable. No real prose yet — just front-matter headers and one-line placeholders.

### Expected Outcomes
The following files exist and appear in the built site nav:

```
docs/
├── index.md
├── 01-fundamentals/
│   ├── learn.md
│   ├── practice.md
│   └── challenges.md
├── 02-web-development/
│   ├── learn.md
│   ├── practice.md
│   └── challenges.md
├── 03-apis/
│   ├── learn.md
│   ├── practice.md
│   └── challenges.md
├── 04-cloud/
│   ├── learn.md
│   ├── practice.md
│   └── challenges.md
├── 05-ai/
│   ├── learn.md
│   ├── practice.md
│   └── challenges.md
└── 06-enterprise-apps/
    ├── learn.md
    ├── practice.md
    └── challenges.md
```

### Todo List
1. Create stub `learn.md`, `practice.md`, `challenges.md` for **01-fundamentals**.
2. Create stub `learn.md`, `practice.md`, `challenges.md` for **02-web-development**.
3. Create stub `learn.md`, `practice.md`, `challenges.md` for **03-apis**.
4. Create stub `learn.md`, `practice.md`, `challenges.md` for **04-cloud**.
5. Create stub `learn.md`, `practice.md`, `challenges.md` for **05-ai**.
6. Create stub `learn.md`, `practice.md`, `challenges.md` for **06-enterprise-apps**.
7. Update `mkdocs.yml` `nav:` block to wire all stubs to the correct paths.

### Relevant Context
- Each stub file needs a front-matter title (`# Track N — Section`) so MkDocs renders a clean page.
- The `nav:` in `mkdocs.yml` must match the exact file paths created here.

---

## Sub-Task 3 — Home Page & Visual Identity

**Status:** `[ ] pending`

### Intent
Build an attractive, on-brand `docs/index.md` landing page and configure the Material theme to use the Rust-themed orange/dark palette with the Rust gear logo and the site tagline.

### Expected Outcomes
- `docs/index.md` contains: hero section with tagline, brief programme overview, cards linking to each track.
- `mkdocs.yml` theme block is finalised with:
  - `primary: deep-orange` (Rust orange closest in Material palette)
  - `accent: orange`
  - `scheme: slate` (dark mode default)
  - Custom logo pointing to the Rust gear SVG.
- `docs/assets/logo.svg` (or `.png`) contains the Rust gear icon.
- A `docs/stylesheets/extra.css` file with any fine-tuning overrides for Rust brand colours (`#CE422B` primary hex).

### Todo List
1. Download / copy the Rust gear SVG into `docs/assets/rust-gear.svg`.
2. Create `docs/stylesheets/extra.css` with CSS variable overrides for `#CE422B` Rust orange.
3. Update `mkdocs.yml` `theme:` block to set palette, logo, favicon, and link the extra stylesheet.
4. Write the full `docs/index.md` landing page with:
   - Hero admonition with tagline.
   - "What You'll Learn" grid cards (one per track).
   - "How It Works" section (Learn → Practice → Challenge loop).
   - "Get Started" button linking to Track 1.

### Relevant Context
- Rust brand colour: `#CE422B` (official Rust orange).
- Material `grid cards` feature requires `attr_list` and `md_in_html` extensions enabled in `mkdocs.yml`.
- Logo path in `mkdocs.yml`: `assets/rust-gear.svg`.

---

## Sub-Task 4 — Content: Track 1 — Fundamentals

**Status:** `[ ] pending`

### Intent
Author the full learning content for Track 1: Rust Fundamentals. This is the most important track — it sets the tone and quality bar for all other tracks.

### Expected Outcomes
- `docs/01-fundamentals/learn.md` covers: variables & mutability, data types, functions, control flow, ownership, borrowing, lifetimes, structs, enums, pattern matching, error handling (`Result`/`Option`), modules, crates.
- `docs/01-fundamentals/practice.md` contains 10–15 incremental exercises with starter code snippets and expected outputs.
- `docs/01-fundamentals/challenges.md` contains 5 harder challenges requiring multi-concept integration.

### Todo List
1. Write `docs/01-fundamentals/learn.md` (concept explanations + code examples).
2. Write `docs/01-fundamentals/practice.md` (guided exercises).
3. Write `docs/01-fundamentals/challenges.md` (challenge problems).

### Relevant Context
- Target audience: complete Rust beginners who know at least one other language.
- All code blocks must use ` ```rust ` fences so Material theme syntax-highlights them.
- Use MkDocs Material admonitions (`!!! tip`, `!!! warning`, `!!! example`) for callouts.

---

## Sub-Task 5 — Content: Track 2 — Web Development

**Status:** `[ ] pending`

### Intent
Author content for Track 2: Rust Web Development — building HTTP servers and web apps with Rust.

### Expected Outcomes
- `docs/02-web-development/learn.md` covers: Axum/Actix-web intro, routing, middleware, templating (Tera/Askama), serving static files, async with Tokio.
- `docs/02-web-development/practice.md`: exercises building REST endpoints, HTML pages, form handling.
- `docs/02-web-development/challenges.md`: 5 challenges (e.g. build a full todo-list web app).

### Todo List
1. Write `docs/02-web-development/learn.md`.
2. Write `docs/02-web-development/practice.md`.
3. Write `docs/02-web-development/challenges.md`.

### Relevant Context
- Primary framework: Axum (with Tokio). Note Actix-web as an alternative.
- Prerequisites: Track 1 complete.

---

## Sub-Task 6 — Content: Track 3 — APIs

**Status:** `[ ] pending`

### Intent
Author content for Track 3: Building and consuming REST / GraphQL APIs in Rust.

### Expected Outcomes
- `docs/03-apis/learn.md`: REST design in Rust, JSON serialisation with `serde`, `reqwest` for HTTP clients, OpenAPI spec generation, GraphQL with `async-graphql`.
- `docs/03-apis/practice.md`: exercises building and calling APIs.
- `docs/03-apis/challenges.md`: 5 challenges (e.g. a full CRUD API with authentication).

### Todo List
1. Write `docs/03-apis/learn.md`.
2. Write `docs/03-apis/practice.md`.
3. Write `docs/03-apis/challenges.md`.

### Relevant Context
- Key crates: `serde`, `serde_json`, `reqwest`, `axum`, `async-graphql`, `utoipa`.
- Prerequisites: Track 2 complete.

---

## Sub-Task 7 — Content: Track 4 — Cloud

**Status:** `[ ] pending`

### Intent
Author content for Track 4: Deploying Rust applications to cloud platforms.

### Expected Outcomes
- `docs/04-cloud/learn.md`: containerising Rust with Docker, deploying to Kubernetes, AWS Lambda with Rust (cargo-lambda), environment config, secrets management, health checks.
- `docs/04-cloud/practice.md`: exercises packaging and deploying apps.
- `docs/04-cloud/challenges.md`: 5 challenges (e.g. serverless Rust function on AWS).

### Todo List
1. Write `docs/04-cloud/learn.md`.
2. Write `docs/04-cloud/practice.md`.
3. Write `docs/04-cloud/challenges.md`.

### Relevant Context
- Key tools: Docker multi-stage builds, `cargo-lambda`, Kubernetes YAML, IBM Cloud / AWS / GCP options.
- Prerequisites: Track 3 complete.

---

## Sub-Task 8 — Content: Track 5 — AI

**Status:** `[ ] pending`

### Intent
Author content for Track 5: Rust for AI/ML — integrating Rust with AI services and building ML pipelines.

### Expected Outcomes
- `docs/05-ai/learn.md`: Rust + ONNX Runtime, calling LLM APIs (OpenAI/watsonx) from Rust, Candle (HuggingFace Rust ML framework), embedding generation, vector databases.
- `docs/05-ai/practice.md`: exercises calling AI APIs and running inference.
- `docs/05-ai/challenges.md`: 5 challenges (e.g. build a Rust CLI chatbot).

### Todo List
1. Write `docs/05-ai/learn.md`.
2. Write `docs/05-ai/practice.md`.
3. Write `docs/05-ai/challenges.md`.

### Relevant Context
- Key crates: `candle-core`, `ort` (ONNX), `async-openai`, `reqwest` for watsonx APIs.
- Prerequisites: Track 3 + 4 complete.

---

## Sub-Task 9 — Content: Track 6 — Enterprise Apps

**Status:** `[ ] pending`

### Intent
Author content for Track 6: Production-grade enterprise Rust — databases, messaging, observability, security, and architecture patterns.

### Expected Outcomes
- `docs/06-enterprise-apps/learn.md`: SQLx/Diesel for databases, migrations, messaging with Kafka/NATS, tracing with `tracing` + OpenTelemetry, auth (JWT, OAuth2), WASM compilation targets, performance tuning.
- `docs/06-enterprise-apps/practice.md`: exercises integrating databases and observability.
- `docs/06-enterprise-apps/challenges.md`: 5 capstone challenges (e.g. microservice with DB + telemetry).

### Todo List
1. Write `docs/06-enterprise-apps/learn.md`.
2. Write `docs/06-enterprise-apps/practice.md`.
3. Write `docs/06-enterprise-apps/challenges.md`.

### Relevant Context
- Key crates: `sqlx`, `diesel`, `rdkafka`, `tracing`, `opentelemetry`, `jsonwebtoken`.
- Prerequisites: All prior tracks.

---

## Sub-Task 10 — Final Polish & Validation

**Status:** `[ ] pending`

### Intent
Validate the full site builds cleanly, all navigation links resolve, and the deployed GitHub Pages URL is live and accessible.

### Expected Outcomes
- `mkdocs build --strict` passes with zero warnings.
- All 19 content pages (1 home + 6 tracks × 3 pages) appear in the nav.
- GitHub Actions workflow runs green on the `main` branch.
- Site is live at `https://practice-rust-for-beginners.github.io/`.
- `README.md` added at repo root with local dev instructions (`pip install -r requirements.txt && mkdocs serve`).

### Todo List
1. Run `mkdocs build --strict` locally (or verify CI passes).
2. Fix any broken nav links or missing file references.
3. Write `README.md` with setup, local dev, and contribution instructions.
4. Verify the live GitHub Pages URL resolves correctly.

### Relevant Context
- `--strict` mode converts MkDocs warnings to errors — catches broken links early.
- GitHub Pages URL pattern for org sites: `https://<org>.github.io/`.
