# Project: CLI Calculator

**Track:** 🦀 Fundamentals &nbsp;|&nbsp; **Difficulty:** 🟢 Beginner &nbsp;|&nbsp; **Crates:** standard library only

---

## What You'll Build

A command-line calculator that evaluates expressions like `10 / 3` or `2 + 2`, handles all common operators, prints results with precision, and maintains a session history you can recall.

```text
$ cargo run
Rust Calculator — type 'help' or 'quit'
> 10 + 5
= 15
> 100 / 7
= 14.285714285714286
> 2 ^ 8
= 256
> history
[1] 10 + 5 = 15
[2] 100 / 7 = 14.285714285714286
[3] 2 ^ 8 = 256
> quit
Bye!
```

---

## Tech Stack

| Component | Choice |
|-----------|--------|
| Argument parsing | `std::env::args` / interactive REPL |
| I/O | `std::io` (stdin/stdout) |
| Error handling | Custom `CalcError` enum |
| History | `Vec<String>` |

---

## Step 1 — Project Setup

```bash
cargo new rust-calculator
cd rust-calculator
```

---

## Step 2 — Define the Operator Enum

```rust
// src/operator.rs
use std::fmt;

#[derive(Debug, Clone, PartialEq)]
pub enum Op {
    Add,
    Sub,
    Mul,
    Div,
    Pow,
    Rem,
}

impl Op {
    pub fn from_str(s: &str) -> Option<Self> {
        match s {
            "+" => Some(Op::Add),
            "-" => Some(Op::Sub),
            "*" => Some(Op::Mul),
            "/" => Some(Op::Div),
            "^" => Some(Op::Pow),
            "%" => Some(Op::Rem),
            _   => None,
        }
    }
}

impl fmt::Display for Op {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        let sym = match self {
            Op::Add => "+", Op::Sub => "-", Op::Mul => "*",
            Op::Div => "/", Op::Pow => "^", Op::Rem => "%",
        };
        write!(f, "{sym}")
    }
}
```

---

## Step 3 — Define the Error Type

```rust
// src/error.rs
use std::fmt;

#[derive(Debug)]
pub enum CalcError {
    DivisionByZero,
    InvalidNumber(String),
    UnknownOperator(String),
    InvalidExpression,
}

impl fmt::Display for CalcError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            CalcError::DivisionByZero        => write!(f, "Error: division by zero"),
            CalcError::InvalidNumber(s)      => write!(f, "Error: '{}' is not a number", s),
            CalcError::UnknownOperator(s)    => write!(f, "Error: unknown operator '{}'", s),
            CalcError::InvalidExpression     => write!(f, "Error: expected '<number> <op> <number>'"),
        }
    }
}
```

---

## Step 4 — The Calculator Core

```rust
// src/calc.rs
use crate::{operator::Op, error::CalcError};

pub fn parse(input: &str) -> Result<(f64, Op, f64), CalcError> {
    let parts: Vec<&str> = input.trim().splitn(3, ' ').collect();
    if parts.len() != 3 {
        return Err(CalcError::InvalidExpression);
    }
    let a = parts[0].parse::<f64>()
        .map_err(|_| CalcError::InvalidNumber(parts[0].to_string()))?;
    let op = Op::from_str(parts[1])
        .ok_or_else(|| CalcError::UnknownOperator(parts[1].to_string()))?;
    let b = parts[2].parse::<f64>()
        .map_err(|_| CalcError::InvalidNumber(parts[2].to_string()))?;
    Ok((a, op, b))
}

pub fn evaluate(a: f64, op: &Op, b: f64) -> Result<f64, CalcError> {
    match op {
        Op::Add => Ok(a + b),
        Op::Sub => Ok(a - b),
        Op::Mul => Ok(a * b),
        Op::Div => {
            if b == 0.0 { Err(CalcError::DivisionByZero) }
            else { Ok(a / b) }
        }
        Op::Pow => Ok(a.powf(b)),
        Op::Rem => {
            if b == 0.0 { Err(CalcError::DivisionByZero) }
            else { Ok(a % b) }
        }
    }
}
```

---

## Step 5 — The REPL Loop

```rust
// src/main.rs
mod operator;
mod error;
mod calc;

use std::io::{self, Write};

fn main() {
    println!("Rust Calculator — type 'help' or 'quit'");
    let mut history: Vec<String> = Vec::new();

    loop {
        print!("> ");
        io::stdout().flush().unwrap();

        let mut input = String::new();
        if io::stdin().read_line(&mut input).is_err() { break; }
        let input = input.trim();

        match input {
            "quit" | "exit" | "q" => { println!("Bye!"); break; }
            "history" | "h" => {
                if history.is_empty() { println!("No history yet."); }
                else {
                    for (i, entry) in history.iter().enumerate() {
                        println!("[{}] {}", i + 1, entry);
                    }
                }
            }
            "clear" => history.clear(),
            "help"  => print_help(),
            ""      => continue,
            expr    => {
                match calc::parse(expr) {
                    Err(e) => println!("{e}"),
                    Ok((a, op, b)) => match calc::evaluate(a, &op, b) {
                        Err(e)      => println!("{e}"),
                        Ok(result)  => {
                            println!("= {result}");
                            history.push(format!("{a} {op} {b} = {result}"));
                        }
                    }
                }
            }
        }
    }
}

fn print_help() {
    println!("Usage: <number> <operator> <number>");
    println!("Operators: + - * / ^ %");
    println!("Commands:  history, clear, help, quit");
}
```

---

## Step 6 — Run It

```bash
cargo run
```

---

## Stretch Goals

- [ ] Support multi-operation expressions: `2 + 3 * 4` with correct precedence
- [ ] Add `ans` keyword that refers to the last result
- [ ] Save history to `~/.calc_history` and reload it on startup
- [ ] Add unit tests for every operator and edge case (`cargo test`)

---

[Back to All Projects](../index.md){ .md-button }


[Next Project: Text Adventure Game](text-adventure.md){ .md-button .md-button--primary }
