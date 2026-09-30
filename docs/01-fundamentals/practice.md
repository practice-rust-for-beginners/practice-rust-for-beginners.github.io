# Track 1 — Fundamentals: Practice

Work through these exercises in order. Each one targets a specific concept from the Learn section. Every exercise gives you a starting point — your job is to make it compile and produce the stated output.

!!! tip "How to work these exercises"
    1. Create a new project: `cargo new exercise_N && cd exercise_N`
    2. Edit `src/main.rs` with the starter code.
    3. Run `cargo run` and compare against the expected output.
    4. Use `cargo check` for a faster feedback loop when you only want type errors.

---

## Exercise 1 — Variables & Shadowing

**Goal:** Understand the difference between `mut` and shadowing.

**Starter code:**
```rust
fn main() {
    let x = 10;
    // TODO: shadow x with its value doubled
    // TODO: print both the original type change and the new value
    println!("x = {x}");
}
```

**Expected output:**
```text
x = 20
```

??? example "Solution"
    ```rust
    fn main() {
        let x = 10;
        let x = x * 2;
        println!("x = {x}");
    }
    ```

---

## Exercise 2 — Type Conversions

**Goal:** Practice casting between numeric types and using `parse`.

**Starter code:**
```rust
fn main() {
    let s = "42";
    // TODO: parse s into an i32, double it, then print it
    // TODO: cast an f64 to u32 and print it
    let f: f64 = 9.99;
}
```

**Expected output:**
```text
84
9
```

??? example "Solution"
    ```rust
    fn main() {
        let s = "42";
        let n: i32 = s.parse().unwrap();
        println!("{}", n * 2);

        let f: f64 = 9.99;
        println!("{}", f as u32);
    }
    ```

---

## Exercise 3 — FizzBuzz

**Goal:** Control flow with `for`, `if/else`, and the modulo operator.

Write a function `fizzbuzz(n: u32) -> String` that returns:
- `"FizzBuzz"` if divisible by both 3 and 5
- `"Fizz"` if divisible by 3
- `"Buzz"` if divisible by 5
- the number as a string otherwise

Print the result for 1 through 20.

**Expected output (excerpt):**
```text
1
2
Fizz
4
Buzz
Fizz
7
8
Fizz
Buzz
11
Fizz
13
14
FizzBuzz
...
```

??? example "Solution"
    ```rust
    fn fizzbuzz(n: u32) -> String {
        match (n % 3, n % 5) {
            (0, 0) => String::from("FizzBuzz"),
            (0, _) => String::from("Fizz"),
            (_, 0) => String::from("Buzz"),
            _      => n.to_string(),
        }
    }

    fn main() {
        for i in 1..=20 { println!("{}", fizzbuzz(i)); }
    }
    ```

---

## Exercise 4 — Ownership Move vs Clone

**Goal:** Observe the compiler error when you use a moved value, then fix it.

```rust
fn takes_ownership(s: String) {
    println!("{s}");
}

fn main() {
    let s = String::from("hello");
    takes_ownership(s);
    // TODO: this line will fail — why?
    println!("{s}");
}
```

Fix the program in **two different ways**:
1. Clone `s` before passing it.
2. Change `takes_ownership` to borrow `&str` instead.

---

## Exercise 5 — Borrowing

**Goal:** Write a function that counts vowels in a string without taking ownership.

```rust
fn count_vowels(s: &str) -> usize {
    // TODO: count 'a', 'e', 'i', 'o', 'u' (case-insensitive)
    todo!()
}

fn main() {
    let sentence = String::from("Hello, Rust World!");
    println!("{}", count_vowels(&sentence));
    println!("{}", sentence); // must still be usable
}
```

**Expected output:**
```text
4
Hello, Rust World!
```

??? example "Solution"
    ```rust
    fn count_vowels(s: &str) -> usize {
        s.chars().filter(|c| "aeiouAEIOU".contains(*c)).count()
    }

    fn main() {
        let sentence = String::from("Hello, Rust World!");
        println!("{}", count_vowels(&sentence));
        println!("{}", sentence);
    }
    ```

---

## Exercise 6 — Structs & Methods

**Goal:** Model a `BankAccount` with deposit, withdraw, and balance.

```rust
struct BankAccount {
    owner:   String,
    balance: f64,
}

impl BankAccount {
    // TODO: fn new(owner: &str, opening_balance: f64) -> Self
    // TODO: fn deposit(&mut self, amount: f64)
    // TODO: fn withdraw(&mut self, amount: f64) -> Result<(), String>
    // TODO: fn balance(&self) -> f64
}

fn main() {
    let mut account = BankAccount::new("Alice", 100.0);
    account.deposit(50.0);
    account.withdraw(30.0).unwrap();
    println!("Balance: {:.2}", account.balance()); // 120.00
    println!("{:?}", account.withdraw(200.0));      // Err("Insufficient funds")
}
```

---

## Exercise 7 — Enums & Match

**Goal:** Build a simple `Command` enum for a text-based game.

```rust
enum Command {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(u8, u8, u8),
}

fn process(cmd: Command) {
    // TODO: match on cmd and print a description of each variant
    todo!()
}

fn main() {
    process(Command::Move { x: 10, y: 25 });
    process(Command::Write(String::from("hello")));
    process(Command::ChangeColor(255, 128, 0));
    process(Command::Quit);
}
```

**Expected output:**
```text
Moving to (10, 25)
Writing: hello
Changing colour to rgb(255, 128, 0)
Quitting
```

---

## Exercise 8 — Option Chaining

**Goal:** Practice `Option` methods without explicit `match`.

```rust
fn find_first_even(numbers: &[i32]) -> Option<i32> {
    // TODO: return the first even number, doubled
    // Hint: use .iter().find() and .map()
    todo!()
}

fn main() {
    println!("{:?}", find_first_even(&[1, 3, 4, 7, 8])); // Some(8)
    println!("{:?}", find_first_even(&[1, 3, 5, 7]));    // None
}
```

??? example "Solution"
    ```rust
    fn find_first_even(numbers: &[i32]) -> Option<i32> {
        numbers.iter().find(|&&n| n % 2 == 0).map(|&n| n * 2)
    }
    ```

---

## Exercise 9 — Error Handling with `?`

**Goal:** Chain fallible operations cleanly.

```rust
use std::num::ParseIntError;

fn sum_strings(a: &str, b: &str) -> Result<i32, ParseIntError> {
    // TODO: parse both strings and return their sum
    // Use the `?` operator — no match or unwrap allowed
    todo!()
}

fn main() {
    println!("{:?}", sum_strings("10", "32"));   // Ok(42)
    println!("{:?}", sum_strings("10", "abc"));  // Err(...)
}
```

---

## Exercise 10 — Generics & Traits

**Goal:** Implement a generic `Stack<T>`.

```rust
struct Stack<T> {
    // TODO: store elements in a Vec<T>
}

impl<T> Stack<T> {
    // TODO: fn new() -> Self
    // TODO: fn push(&mut self, item: T)
    // TODO: fn pop(&mut self) -> Option<T>
    // TODO: fn peek(&self) -> Option<&T>
    // TODO: fn is_empty(&self) -> bool
}

fn main() {
    let mut stack: Stack<i32> = Stack::new();
    stack.push(1);
    stack.push(2);
    stack.push(3);
    println!("{:?}", stack.peek());  // Some(3)
    println!("{:?}", stack.pop());   // Some(3)
    println!("{:?}", stack.pop());   // Some(2)
}
```

---

## Exercise 11 — Iterators & Closures

**Goal:** Use iterator adaptors to process a list of scores.

```rust
fn main() {
    let scores = vec![85, 42, 91, 67, 55, 78, 94, 33, 70, 88];

    // TODO: using iterator methods only (no loops):
    // 1. Filter scores >= 60
    // 2. Map each to a letter grade (>=90 "A", >=80 "B", >=70 "C", else "D")
    // 3. Collect into a Vec<&str> and print it
}
```

**Expected output:**
```text
["B", "A", "D", "C", "A", "C", "B"]
```

---

## Exercise 12 — Lifetimes

**Goal:** Fix the lifetime annotations in this function.

```rust
// This does NOT compile — add the correct lifetime annotations
fn longest_word(sentence1: &str, sentence2: &str) -> &str {
    let w1 = sentence1.split_whitespace().max_by_key(|w| w.len()).unwrap_or("");
    let w2 = sentence2.split_whitespace().max_by_key(|w| w.len()).unwrap_or("");
    if w1.len() >= w2.len() { w1 } else { w2 }
}

fn main() {
    let s1 = String::from("the quick brown fox");
    let s2 = String::from("jumps over a lazy dog");
    println!("{}", longest_word(&s1, &s2)); // "jumps" or "quick" or "brown"
}
```

---

## Exercise 13 — Modules

**Goal:** Organise the `BankAccount` from Exercise 6 into its own module.

Create a project with:
```text
src/
  main.rs
  bank.rs   ← put BankAccount here
```

`main.rs` should use `mod bank;` and `use bank::BankAccount;` to access it.

---

[Go to Challenges](challenges.md){ .md-button .md-button--primary }
[Back to Learn](learn.md){ .md-button }
