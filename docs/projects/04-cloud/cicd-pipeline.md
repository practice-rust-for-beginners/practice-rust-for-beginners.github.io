# Project: CI/CD Pipeline

**Track:** ☁️ Cloud &nbsp;|&nbsp; **Difficulty:** 🔴 Advanced &nbsp;|&nbsp; **Tools:** GitHub Actions, Docker, kubectl

---

## What You'll Build

A complete, multi-stage GitHub Actions CI/CD pipeline for a Rust web service. Every pull request runs linting and tests; every push to `main` deploys to staging; every version tag deploys to production and creates a GitHub Release.

```text
PR opened  → lint + test + build + docker build (no push)
push main  → lint + test + build + push image → deploy staging
tag v*.*.*  → promote image → deploy production + GitHub Release
```

---

## Pipeline Architecture

```text
┌─────────────┐     ┌──────────────┐     ┌──────────────────┐
│  CI (PR)    │     │  Staging CD  │     │  Production CD   │
│             │     │  (main push) │     │  (version tag)   │
│ fmt check   │     │  build+push  │     │  tag+push image  │
│ clippy      │     │  image:sha   │     │  image:v1.2.3    │
│ test        │     │  k8s deploy  │     │  k8s deploy prod │
│ build check │     │  staging ns  │     │  GitHub Release  │
└─────────────┘     └──────────────┘     └──────────────────┘
```

---

## Step 1 — Repository Secrets

In **GitHub → Settings → Secrets and variables → Actions**, add:

| Secret | Value |
|--------|-------|
| `REGISTRY_USERNAME` | Docker Hub username |
| `REGISTRY_PASSWORD` | Docker Hub token |
| `KUBECONFIG_STAGING` | base64-encoded kubeconfig for staging cluster |
| `KUBECONFIG_PROD`    | base64-encoded kubeconfig for production cluster |

---

## Step 2 — Reusable Build Workflow

```yaml
# .github/workflows/build.yml
name: Build & Test

on:
  workflow_call:
    outputs:
      image_tag:
        value: ${{ jobs.build.outputs.image_tag }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy, rustfmt

      - name: Cache Cargo
        uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/git
            target
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}

      - name: Format check
        run: cargo fmt --all -- --check

      - name: Clippy
        run: cargo clippy -- -D warnings

      - name: Test
        run: cargo test

      - name: Build release
        run: cargo build --release
```

---

## Step 3 — CI Workflow (Pull Requests)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  test:
    uses: ./.github/workflows/build.yml

  docker-build-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: false
          tags: myapp:pr-check
```

---

## Step 4 — Staging Deploy (push to main)

```yaml
# .github/workflows/deploy-staging.yml
name: Deploy Staging

on:
  push:
    branches: [main]

jobs:
  build:
    uses: ./.github/workflows/build.yml

  push-image:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            myorg/myapp:latest
            myorg/myapp:${{ github.sha }}

  deploy:
    needs: push-image
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Set up kubeconfig
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBECONFIG_STAGING }}" | base64 -d > ~/.kube/config

      - name: Deploy to staging
        run: |
          kubectl set image deployment/myapp \
            myapp=myorg/myapp:${{ github.sha }} \
            -n staging
          kubectl rollout status deployment/myapp -n staging --timeout=120s
```

---

## Step 5 — Production Deploy (version tag)

```yaml
# .github/workflows/deploy-production.yml
name: Deploy Production

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  tag-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}

      - name: Retag staging image as version
        run: |
          docker pull myorg/myapp:latest
          docker tag  myorg/myapp:latest myorg/myapp:${{ github.ref_name }}
          docker push myorg/myapp:${{ github.ref_name }}

  deploy-prod:
    needs: tag-and-push
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Set up kubeconfig
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBECONFIG_PROD }}" | base64 -d > ~/.kube/config

      - name: Deploy to production
        run: |
          kubectl set image deployment/myapp \
            myapp=myorg/myapp:${{ github.ref_name }} \
            -n production
          kubectl rollout status deployment/myapp -n production --timeout=180s

  release:
    needs: deploy-prod
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Generate changelog
        id: changelog
        run: |
          PREV=$(git tag --sort=-version:refname | sed -n '2p')
          echo "log=$(git log ${PREV}..HEAD --oneline | head -20 | sed 's/"/\\"/g')" >> $GITHUB_OUTPUT

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          body: |
            ## What's Changed
            ${{ steps.changelog.outputs.log }}
```

---

## Step 6 — Trigger It

```bash
# Trigger CI
git checkout -b my-feature
git push origin my-feature
# Open a PR → CI runs automatically

# Trigger staging deploy
git checkout main
git merge my-feature
git push origin main

# Trigger production deploy
git tag v1.2.0
git push origin v1.2.0
```

---

## Stretch Goals

- [ ] Add Slack notifications on deploy success/failure
- [ ] Add smoke tests that run against staging after deploy
- [ ] Add security scanning with `cargo audit` in the CI step
- [ ] Add SAST scanning with `trivy` on the Docker image

---

[Back to All Projects](../index.md){ .md-button }


[Next Track: AI Projects](../05-ai/cli-chatbot.md){ .md-button .md-button--primary }
