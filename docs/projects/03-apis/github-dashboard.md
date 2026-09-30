# Project: GitHub Dashboard

**Track:** 🔌 APIs &nbsp;|&nbsp; **Difficulty:** 🟡 Intermediate &nbsp;|&nbsp; **Crates:** reqwest, serde, tokio

---

## What You'll Build

A terminal dashboard that fetches your GitHub profile, repositories, open PRs, and recent activity — all using the GitHub REST API from Rust.

```text
$ cargo run -- --user ferris-the-crab --token ghp_xxxxx

╔══════════════════════════════════════╗
║  ferris-the-crab (@ferris-the-crab)  ║
║  Followers: 142  Following: 38       ║
╚══════════════════════════════════════╝

Top Repositories (by stars):
  ★ 1,204  rust-crab-lib        Rust utilities for crustaceans
  ★   847  awesome-ferris       A curated list of Ferris resources
  ★   312  crab-web             A tiny Axum web framework wrapper

Open Pull Requests (last 5):
  #42  [rust-crab-lib]  Add serde support for CrabId
  #39  [crab-web]       Fix CORS middleware ordering

Recent Activity (last 7 days):
  PushEvent      rust-crab-lib        2024-06-12
  CreateEvent    awesome-ferris       2024-06-11
```

---

## Tech Stack

```toml
[dependencies]
reqwest   = { version = "0.12", features = ["json"] }
tokio     = { version = "1",    features = ["full"] }
serde     = { version = "1",    features = ["derive"] }
serde_json = "1"
chrono    = { version = "0.4",  features = ["serde"] }
```

---

## Step 1 — GitHub API Client

```rust
// src/github.rs
use reqwest::{Client, header::{AUTHORIZATION, USER_AGENT, HeaderMap, HeaderValue}};
use serde::Deserialize;
use chrono::{DateTime, Utc};

pub struct GitHubClient {
    client:   Client,
    base_url: &'static str,
}

impl GitHubClient {
    pub fn new(token: &str) -> Self {
        let mut headers = HeaderMap::new();
        headers.insert(USER_AGENT, HeaderValue::from_static("rust-github-dashboard"));
        headers.insert(AUTHORIZATION,
            HeaderValue::from_str(&format!("Bearer {token}")).unwrap());
        Self {
            client:   Client::builder().default_headers(headers).build().unwrap(),
            base_url: "https://api.github.com",
        }
    }

    pub async fn user(&self, username: &str) -> reqwest::Result<GitHubUser> {
        self.client.get(format!("{}/users/{username}", self.base_url))
            .send().await?.json().await
    }

    pub async fn repos(&self, username: &str) -> reqwest::Result<Vec<Repo>> {
        self.client.get(format!("{}/users/{username}/repos?per_page=100&sort=stars", self.base_url))
            .send().await?.json().await
    }

    pub async fn open_prs(&self, username: &str) -> reqwest::Result<Vec<PullRequest>> {
        self.client.get(format!("{}/search/issues?q=author:{username}+is:pr+is:open", self.base_url))
            .send().await?.json::<SearchResult<PullRequest>>().await
            .map(|r| r.items)
    }

    pub async fn events(&self, username: &str) -> reqwest::Result<Vec<Event>> {
        self.client.get(format!("{}/users/{username}/events?per_page=30", self.base_url))
            .send().await?.json().await
    }
}
```

---

## Step 2 — Response Types

```rust
// src/github.rs (continued)
#[derive(Deserialize)]
pub struct GitHubUser {
    pub login:      String,
    pub name:       Option<String>,
    pub followers:  u32,
    pub following:  u32,
    pub public_repos: u32,
    pub bio:        Option<String>,
}

#[derive(Deserialize)]
pub struct Repo {
    pub name:             String,
    pub description:      Option<String>,
    pub stargazers_count: u32,
    pub language:         Option<String>,
    pub fork:             bool,
}

#[derive(Deserialize)]
pub struct PullRequest {
    pub number:     u32,
    pub title:      String,
    pub repository_url: String,
}

#[derive(Deserialize)]
pub struct SearchResult<T> {
    pub items: Vec<T>,
}

#[derive(Deserialize)]
pub struct Event {
    #[serde(rename = "type")]
    pub kind:       String,
    pub repo:       EventRepo,
    pub created_at: DateTime<Utc>,
}

#[derive(Deserialize)]
pub struct EventRepo { pub name: String }
```

---

## Step 3 — Display Logic

```rust
// src/display.rs
use crate::github::{GitHubUser, Repo, PullRequest, Event};

pub fn print_user(user: &GitHubUser) {
    let name = user.name.as_deref().unwrap_or(&user.login);
    println!("\n╔{:═<40}╗", "");
    println!("║  {:38}║", format!("{} (@{})", name, user.login));
    println!("║  {:38}║", format!("Followers: {}  Following: {}", user.followers, user.following));
    println!("╚{:═<40}╝\n", "");
}

pub fn print_repos(repos: &[Repo]) {
    println!("Top Repositories (by stars):");
    for r in repos.iter().filter(|r| !r.fork).take(5) {
        println!("  ★ {:5}  {:<24} {}",
            r.stargazers_count,
            r.name,
            r.description.as_deref().unwrap_or(""));
    }
    println!();
}

pub fn print_prs(prs: &[PullRequest]) {
    println!("Open Pull Requests (last 5):");
    for pr in prs.iter().take(5) {
        let repo = pr.repository_url.split('/').last().unwrap_or("?");
        println!("  #{:<5} [{repo}]  {}", pr.number, pr.title);
    }
    println!();
}

pub fn print_events(events: &[Event]) {
    println!("Recent Activity (last 7 days):");
    for e in events.iter().take(10) {
        println!("  {:<20} {:<28} {}",
            e.kind, e.repo.name,
            e.created_at.format("%Y-%m-%d"));
    }
}
```

---

## Step 4 — Main

```rust
// src/main.rs
mod github;
mod display;

use github::GitHubClient;

#[tokio::main]
async fn main() {
    let mut args = std::env::args().skip(1);
    let username = args.find(|a| !a.starts_with('-')).expect("Usage: --user <name>");
    let token_flag = std::env::args().skip_while(|a| a != "--token").nth(1);
    let token = token_flag.or_else(|| std::env::var("GITHUB_TOKEN").ok())
        .expect("Provide --token or set GITHUB_TOKEN env var");

    let client = GitHubClient::new(&token);

    let (user, mut repos, prs, events) = tokio::try_join!(
        client.user(&username),
        client.repos(&username),
        client.open_prs(&username),
        client.events(&username),
    ).expect("GitHub API error");

    repos.sort_by(|a, b| b.stargazers_count.cmp(&a.stargazers_count));

    display::print_user(&user);
    display::print_repos(&repos);
    display::print_prs(&prs);
    display::print_events(&events);
}
```

---

## Step 5 — Run It

```bash
export GITHUB_TOKEN=ghp_your_token_here
cargo run -- ferris-the-crab
```

---

## Stretch Goals

- [ ] Add `--org <name>` to show organisation repositories and members
- [ ] Export to JSON or Markdown file with `--output dashboard.md`
- [ ] Add coloured terminal output using the `colored` crate
- [ ] Cache responses for 5 minutes to avoid API rate limits

---

[Back to All Projects](../index.md){ .md-button }


[Next Project: Notification Service](notification-service.md){ .md-button .md-button--primary }
