# Track 1 — Fundamentals: Challenges

These challenges are intentionally open-ended. There is no single correct solution. The goal is to combine multiple concepts from the Learn section into a working program.

!!! warning "No hand-holding"
    No starter code, no hints unless explicitly given. Use the Rust docs at [doc.rust-lang.org](https://doc.rust-lang.org/std/) and the compiler's error messages — they are excellent teachers.

---

## Challenge 1 — Mini Calculator

Build a command-line calculator that:

1. Reads two numbers and an operator (`+`, `-`, `*`, `/`, `%`) from command-line arguments (`std::env::args`).
2. Returns the correct result as a `f64`.
3. Returns a descriptive error if the operator is unknown or if division by zero is attempted.
4. Exits with code `1` on error, `0` on success.

**Usage:**
```bash
cargo run -- 10 / 3    # → 3.3333333333333335
cargo run -- 10 / 0    # → Error: division by zero
cargo run -- 10 ^ 3    # → Error: unknown operator '^'
```

**Constraints:**
- Model the operator as an `enum Op`.
- Parse arguments using a function that returns `Result<(f64, Op, f64), String>`.
- Use `match` on the `Op` enum to compute the result.

---

## Challenge 2 — Word Frequency Counter

Write a program that:

1. Accepts a multi-line string literal (or reads from stdin — bonus).
2. Counts how many times each word appears (case-insensitive, strip punctuation).
3. Prints the top 10 most frequent words in descending order of frequency.

**Example input:**
```text
the quick brown fox jumps over the lazy dog
the dog barked at the fox
```

**Example output:**
```text
the   : 5
fox   : 2
dog   : 2
quick : 1
brown : 1
...
```

**Constraints:**
- Use `HashMap<String, usize>` for counting.
- Sort using `.sort_by(|a, b| b.1.cmp(&a.1))`.
- Strip punctuation with `.chars().filter(|c| c.is_alphanumeric())`.

---

## Challenge 3 — Linked List

Implement a singly-linked list using Rust's ownership model:

```text
List: 1 → 2 → 3 → Nil
```

Requirements:
- Define a recursive `enum List<T>` with `Cons(T, Box<List<T>>)` and `Nil` variants.
- Implement methods: `push_front`, `pop_front`, `len`, `to_vec`.
- Implement `Display` for `List<T>` where `T: Display`, showing `1 -> 2 -> 3 -> Nil`.

!!! tip "Box<T>"
    You need `Box<T>` to make the recursive type have a known size. `Box<T>` is a heap-allocated smart pointer — it's the simplest of Rust's smart pointers.

---

## Challenge 4 — Shape Calculator with Traits

Design a system for computing properties of 2D shapes:

1. Define a `Shape` trait with methods: `area(&self) -> f64`, `perimeter(&self) -> f64`, `name(&self) -> &str`.
2. Implement `Shape` for: `Circle`, `Rectangle`, `Triangle` (given three sides).
3. Write a function `largest_area(shapes: &[Box<dyn Shape>]) -> &dyn Shape` that returns the shape with the largest area.
4. Print a summary table for a mixed collection of shapes.

**Example output:**
```text
Shape       | Area     | Perimeter
------------|----------|----------
Circle r=5  | 78.54    | 31.42
Rect 4×6    | 24.00    | 20.00
Triangle    | 6.00     | 12.00

Largest: Circle r=5 (area = 78.54)
```

---

## Challenge 5 — File-based Todo List

Build a persistent todo-list CLI that reads and writes a JSON-like plain-text file:

**Commands:**
```bash
cargo run -- add "Buy milk"
cargo run -- add "Write Rust code"
cargo run -- done 1
cargo run -- list
cargo run -- remove 2
```

**`todos.txt` format** (you design it):
```text
1 [] Buy milk
2 [x] Write Rust code
```

Requirements:
- Model a `Todo` struct with `id: u32`, `text: String`, `done: bool`.
- Read/write the file using `std::fs`.
- Parse/serialise using only `std::io` and string manipulation (no external crates).
- Handle all errors gracefully — no `unwrap` in `main`.
- Use modules: `todo.rs` (data model + file I/O), `main.rs` (CLI parsing).

!!! tip "Stretch goal"
    Add `cargo run -- edit <id> "New text"` to rename a todo item.

---

[Back to Practice](practice.md){ .md-button }

[Next Track: Web Development](../02-web-development/learn.md){ .md-button .md-button--primary }
