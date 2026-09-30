# Track 3 — APIs: Practice

Build, consume, and document real APIs. Each exercise reinforces a concept from the Learn section.

---

## Exercise 1 — Serde Roundtrip

Given this JSON string, deserialise it into a Rust struct and re-serialise it:

```json
{
  "id": 42,
  "username": "ferris",
  "roles": ["admin", "editor"],
  "profile": { "bio": "I love crabs", "avatar_url": null }
}
```

Requirements:
- Define nested structs (`User`, `Profile`).
- `avatar_url` must be `Option<String>` and omitted from serialisation when `null`.
- Print the round-tripped JSON.

---

## Exercise 2 — Consuming a Public REST API

Use `reqwest` to query the GitHub API:

1. `GET https://api.github.com/repos/rust-lang/rust` — deserialise and print: repo name, stars, open issues, language.
2. `GET https://api.github.com/repos/rust-lang/rust/releases?per_page=5` — print the last 5 Rust release tag names and their published dates.

!!! tip
    GitHub API requires a `User-Agent` header. Use `.header("User-Agent", "my-app")`.

---

## Exercise 3 — Full CRUD REST API

Build a `books` API backed by an in-memory `HashMap<u32, Book>`:

```rust
#[derive(Clone, Serialize, Deserialize)]
struct Book { id: u32, title: String, author: String, year: u16 }
```

Endpoints:
- `GET /books?page=1&per_page=10` — paginated list
- `GET /books/:id`
- `POST /books`
- `PUT /books/:id`
- `DELETE /books/:id`

All errors return JSON `{"error": "..."}` with the correct status code.

Test every endpoint with `curl` or [Bruno](https://www.usebruno.com/).

---

## Exercise 4 — OpenAPI + Swagger UI

Annotate the books API from Exercise 3 with `utoipa` and serve the Swagger UI at `/swagger-ui`.

The spec must include:
- Schema definition for `Book`.
- Request/response documentation for every endpoint.
- A `tags` group named `"books"`.

---

## Exercise 5 — JWT Authentication

Extend the books API with authentication:

1. `POST /auth/login` accepts `{"username": "admin", "password": "secret"}` and returns a JWT on success.
2. All `/books` routes require a valid `Authorization: Bearer <token>` header.
3. Return `401 Unauthorized` for missing or invalid tokens.

Implement the JWT check as an Axum middleware layer.

---

## Exercise 6 — GraphQL Queries

Build a GraphQL endpoint for the books data:

- Query `book(id: Int!): Book`
- Query `books(author: String): [Book]` — optional filter by author
- Serve GraphiQL playground at `/graphiql`.

---

## Exercise 7 — API Client Library

Write a Rust library crate `books_client` that:
- Wraps `reqwest` to call your books API.
- Exposes async functions: `list_books`, `get_book`, `create_book`, `delete_book`.
- Handles errors with a custom `ClientError` enum.

Write a `main.rs` that uses the client to create 3 books, list them, then delete one.

---

[Go to Challenges :fontawesome-solid-trophy:](challenges.md){ .md-button .md-button--primary }
[Back to Learn :fontawesome-solid-book-open:](learn.md){ .md-button }
