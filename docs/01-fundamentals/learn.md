# Track 1 — Fundamentals: Learn

Welcome to Track 1. This is where your Rust journey begins. By the end of this track you will have a solid mental model of the features that make Rust unique: the ownership system, the type system, and the expressive pattern matching — the foundation everything else is built on.

---

## 1. Hello, World!

```rust
fn main() {
    println!("Hello, world!");
}
```

`fn` declares a function. `main` is the entry point every Rust binary must have. `println!` is a *macro* (the `!` tells you so) that writes to standard output with a newline.

Compile and run with:

```bash
cargo new hello && cd hello
cargo run
```

---

## 2. Variables & Mutability

By default every variable in Rust is **immutable**.

```rust
fn main() {
    let x = 5;
    // x = 6; // ← compile error: cannot assign twice to immutable variable
    
    let mut y = 5;
    y = 6; // ✓ — declared with `mut`
    
    println!("x = {x}, y = {y}");
}
```

!!! tip "Why immutable by default?"
    Immutability is the safest default. The compiler forces you to be explicit when you need mutation, which prevents a whole class of bugs.

**Shadowing** — re-declaring a variable with `let` in the same scope:

```rust
fn main() {
    let spaces = "   ";
    let spaces = spaces.len(); // shadows the previous `spaces`
    println!("{spaces}");      // prints 3
}
```

---

## 3. Data Types

### Scalar types

| Category | Types |
|----------|-------|
| Integer  | `i8`, `i16`, `i32`, `i64`, `i128`, `isize`, `u8`…`usize` |
| Float    | `f32`, `f64` |
| Boolean  | `bool` (`true` / `false`) |
| Char     | `char` (4-byte Unicode scalar value) |

```rust
fn main() {
    let n: i32   = -42;
    let f: f64   = 3.14;
    let b: bool  = true;
    let c: char  = '🦀';
    println!("{n} {f} {b} {c}");
}
```

### Compound types

```rust
fn main() {
    // Tuple — fixed length, mixed types
    let tup: (i32, f64, bool) = (500, 6.4, true);
    let (x, y, z) = tup;          // destructure
    println!("{x} {} {z}", tup.1); // index with .0, .1, …

    // Array — fixed length, same type, stack-allocated
    let months = ["Jan","Feb","Mar","Apr","May","Jun",
                  "Jul","Aug","Sep","Oct","Nov","Dec"];
    println!("{}", months[0]);
}
```

---

## 4. Functions

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b  // no semicolon — this is an expression (the return value)
}

fn greet(name: &str) {        // returns () implicitly
    println!("Hello, {name}!");
}

fn main() {
    greet("Ferris");
    println!("{}", add(3, 4));
}
```

!!! info "Expressions vs. statements"
    - A **statement** performs an action and does not return a value: `let x = 5;`
    - An **expression** evaluates to a value: `x + 1`, `{ let y = 3; y + 1 }`
    - The last expression in a function body (without `;`) is the return value.

---

## 5. Control Flow

### `if` / `else if` / `else`

```rust
fn classify(n: i32) -> &'static str {
    if n < 0      { "negative" }
    else if n == 0 { "zero" }
    else          { "positive" }
}
```

`if` is an expression — assign its result directly:

```rust
let label = if score >= 90 { "A" } else { "B" };
```

### Loops

```rust
fn main() {
    // `loop` — infinite loop; break with a value
    let mut counter = 0;
    let result = loop {
        counter += 1;
        if counter == 10 { break counter * 2; }
    };

    // `while`
    let mut n = 3;
    while n != 0 { println!("{n}"); n -= 1; }

    // `for` — the preferred loop in Rust
    for element in [10, 20, 30] {
        println!("{element}");
    }
    for i in 0..5 { print!("{i} "); }   // 0 1 2 3 4
    for i in 0..=5 { print!("{i} "); }  // 0 1 2 3 4 5
}
```

---

## 6. Ownership — Rust's Superpower

Ownership is **the** feature that makes Rust memory-safe without a garbage collector.

**Three rules:**

1. Each value in Rust has a single **owner**.
2. There can only be **one owner at a time**.
3. When the owner goes out of scope, the value is **dropped** (memory freed).

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;  // s1 is MOVED into s2 — s1 is no longer valid

    // println!("{s1}"); // ← compile error: value borrowed after move

    let s3 = s2.clone(); // explicit deep copy — both valid
    println!("{s2} {s3}");
}
```

!!! warning "Stack vs. Heap"
    Simple scalar types like `i32` implement the `Copy` trait — assignment copies the value on the stack. `String` stores data on the heap, so assignment *moves* ownership instead of copying.

---

## 7. References & Borrowing

Instead of taking ownership, you can **borrow** a value via a reference:

```rust
fn length(s: &String) -> usize {  // borrows s, does not own it
    s.len()
}

fn main() {
    let s = String::from("hello");
    let len = length(&s);           // pass a reference
    println!("'{s}' has {len} chars"); // s still valid
}
```

**Mutable references:**

```rust
fn append(s: &mut String) {
    s.push_str(", world");
}

fn main() {
    let mut s = String::from("hello");
    append(&mut s);
    println!("{s}");
}
```

**The borrowing rules (enforced at compile time):**

- You can have **any number of immutable** references OR **exactly one mutable** reference — never both simultaneously.
- References must always be **valid** (no dangling pointers).

---

## 8. The Slice Type

A slice is a reference to a contiguous sequence of elements:

```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    for (i, &byte) in bytes.iter().enumerate() {
        if byte == b' ' { return &s[0..i]; }
    }
    &s[..]
}

fn main() {
    let sentence = String::from("hello world");
    let word = first_word(&sentence);
    println!("{word}"); // "hello"
}
```

---

## 9. Structs

```rust
#[derive(Debug)]
struct User {
    username: String,
    email:    String,
    active:   bool,
}

fn new_user(email: String, username: String) -> User {
    User { username, email, active: true } // field init shorthand
}

fn main() {
    let mut user1 = new_user(
        String::from("ferris@rust.org"),
        String::from("ferris"),
    );
    user1.active = false;

    // Struct update syntax
    let user2 = User {
        email: String::from("other@example.com"),
        ..user1  // remaining fields from user1
    };
    println!("{user2:?}");
}
```

### Methods

```rust
#[derive(Debug)]
struct Rectangle { width: f64, height: f64 }

impl Rectangle {
    fn area(&self) -> f64 { self.width * self.height }

    fn is_square(&self) -> bool { self.width == self.height }

    fn new(width: f64, height: f64) -> Self { // associated function
        Self { width, height }
    }
}

fn main() {
    let r = Rectangle::new(10.0, 5.0);
    println!("Area: {}, square: {}", r.area(), r.is_square());
}
```

---

## 10. Enums & Pattern Matching

```rust
#[derive(Debug)]
enum Shape {
    Circle(f64),
    Rectangle(f64, f64),
    Triangle { base: f64, height: f64 },
}

fn area(shape: &Shape) -> f64 {
    match shape {
        Shape::Circle(r)                      => std::f64::consts::PI * r * r,
        Shape::Rectangle(w, h)               => w * h,
        Shape::Triangle { base, height }      => 0.5 * base * height,
    }
}

fn main() {
    let shapes = vec![
        Shape::Circle(3.0),
        Shape::Rectangle(4.0, 5.0),
        Shape::Triangle { base: 6.0, height: 8.0 },
];
    for s in &shapes {
        println!("{s:?} → area = {:.2}", area(s));
    }
}
```

### `Option<T>` — the null-free alternative

```rust
fn divide(a: f64, b: f64) -> Option<f64> {
    if b == 0.0 { None } else { Some(a / b) }
}

fn main() {
    match divide(10.0, 2.0) {
        Some(result) => println!("Result: {result}"),
        None         => println!("Cannot divide by zero"),
    }

    // Shorthand with `if let`
    if let Some(v) = divide(9.0, 3.0) {
        println!("{v}");
    }
}
```

---

## 11. Error Handling with `Result<T, E>`

```rust
use std::num::ParseIntError;

fn parse_and_double(s: &str) -> Result<i32, ParseIntError> {
    let n = s.trim().parse::<i32>()?;  // `?` propagates the error
    Ok(n * 2)
}

fn main() {
    match parse_and_double("21") {
        Ok(v)  => println!("Got {v}"),
        Err(e) => println!("Error: {e}"),
    }

    // `unwrap_or` and `expect` for quick prototyping
    let v = parse_and_double("bad").unwrap_or(0);
    println!("{v}"); // 0
}
```

!!! tip "The `?` operator"
    `?` is syntactic sugar: if the `Result` is `Ok(v)` it unwraps to `v`; if it is `Err(e)` it returns early from the current function with that error. Use it to write clean, linear error-handling code.

---

## 12. Generics

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest { largest = item; }
    }
    largest
}

fn main() {
    println!("{}", largest(&[34, 50, 25, 100, 65]));
    println!("{}", largest(&[1.5, 3.14, 2.72]));
}
```

---

## 13. Traits

A **trait** defines shared behaviour — similar to interfaces in other languages.

```rust
trait Summary {
    fn summarise(&self) -> String;
    fn preview(&self) -> String {         // default implementation
        format!("{}…", &self.summarise()[..50.min(self.summarise().len())])
    }
}

struct Article { title: String, author: String, content: String }

impl Summary for Article {
    fn summarise(&self) -> String {
        format!("{}, by {}", self.title, self.author)
    }
}

fn notify(item: &impl Summary) {           // trait bound
    println!("Breaking news! {}", item.summarise());
}
```

---

## 14. Lifetimes

Lifetimes ensure references are always valid.

```rust
// The compiler needs to know how long the returned reference lives.
// `'a` says: the output lives at least as long as both inputs.
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let s1 = String::from("long string");
    let result;
    {
        let s2 = String::from("xyz");
        result = longest(s1.as_str(), s2.as_str());
        println!("Longest: {result}");
    }
}
```

!!! info "Most lifetimes are inferred"
    The compiler performs **lifetime elision** — in the majority of functions you never write lifetime annotations. You only need them when the compiler cannot figure out the relationships itself.

---

## 15. Modules & Crates

```rust
// src/lib.rs
pub mod geometry {
    pub struct Circle { pub radius: f64 }

    impl Circle {
        pub fn new(radius: f64) -> Self { Self { radius } }
        pub fn area(&self) -> f64 { std::f64::consts::PI * self.radius.powi(2) }
    }

    pub mod utils {
        pub fn circumference(r: f64) -> f64 { 2.0 * std::f64::consts::PI * r }
    }
}
```

```rust
// src/main.rs
use my_crate::geometry::{Circle, utils};

fn main() {
    let c = Circle::new(5.0);
    println!("Area: {:.2}", c.area());
    println!("Circumference: {:.2}", utils::circumference(5.0));
}
```

---

## What's Next?

You now have the complete Rust fundamentals toolkit. Consolidate these concepts in the **Practice** section, then test yourself with **Challenges**.

[Go to Practice](practice.md){ .md-button .md-button--primary }
[Jump to Challenges](challenges.md){ .md-button }
