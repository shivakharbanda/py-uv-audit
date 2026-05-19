# Rust Patterns Reference

Detailed code examples for common Rust patterns. Load this file when writing
non-trivial code involving smart pointers, concurrency, generics, or error handling.

---

## Table of Contents
1. [Error Handling](#error-handling)
2. [Ownership & Borrowing](#ownership--borrowing)
3. [Traits & Generics](#traits--generics)
4. [Smart Pointers](#smart-pointers)
5. [Concurrency](#concurrency)
6. [Iterators & Closures](#iterators--closures)
7. [Structs & Enums](#structs--enums)
8. [Modules & Visibility](#modules--visibility)
9. [Testing](#testing)
10. [Common Crates](#common-crates)

---

## Error Handling

### Library crate — typed errors with `thiserror`
```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum DatabaseError {
    #[error("connection failed: {0}")]
    Connection(#[from] std::io::Error),
    #[error("record not found: id={id}")]
    NotFound { id: u64 },
    #[error("query error: {0}")]
    Query(String),
}

pub fn find_user(id: u64) -> Result<User, DatabaseError> {
    if id == 0 {
        return Err(DatabaseError::NotFound { id });
    }
    // ...
    Ok(User { id, name: "Alice".into() })
}
```

### Application — ergonomic with `anyhow`
```rust
use anyhow::{Context, Result};

fn run() -> Result<()> {
    let config = std::fs::read_to_string("config.toml")
        .context("failed to read config.toml")?;
    let port: u16 = config.trim().parse()
        .context("config must contain a port number")?;
    println!("Listening on port {port}");
    Ok(())
}

fn main() {
    if let Err(e) = run() {
        eprintln!("Error: {e:#}");  // {:#} prints the full chain
        std::process::exit(1);
    }
}
```

### Converting between error types
```rust
// Implement From<LibError> for AppError automatically via #[from]
// or manually:
impl From<std::num::ParseIntError> for AppError {
    fn from(e: std::num::ParseIntError) -> Self {
        AppError::Parse(e.to_string())
    }
}
```

---

## Ownership & Borrowing

### Prefer borrows in function parameters
```rust
// ✅ Idiomatic
fn greet(name: &str) { println!("Hello, {name}!"); }
fn sum(numbers: &[i32]) -> i32 { numbers.iter().sum() }
fn open(path: &std::path::Path) -> std::io::Result<()> { todo!() }

// ❌ Overly restrictive
fn greet_bad(name: &String) {}
fn sum_bad(numbers: &Vec<i32>) -> i32 { 0 }
```

### Cow for conditionally owned data
```rust
use std::borrow::Cow;

fn ensure_uppercase(s: &str) -> Cow<'_, str> {
    if s.chars().all(char::is_uppercase) {
        Cow::Borrowed(s)       // no allocation
    } else {
        Cow::Owned(s.to_uppercase())  // allocates only when needed
    }
}
```

### Interior mutability
```rust
use std::cell::RefCell;
use std::rc::Rc;

// Single-threaded shared mutable state
let shared: Rc<RefCell<Vec<i32>>> = Rc::new(RefCell::new(vec![]));
let clone = Rc::clone(&shared);

shared.borrow_mut().push(1);
println!("{:?}", clone.borrow());  // [1]
```

---

## Traits & Generics

### Defining and implementing traits
```rust
pub trait Summary {
    fn summarize_author(&self) -> String;

    // Default implementation
    fn summarize(&self) -> String {
        format!("(Read more from {}...)", self.summarize_author())
    }
}

pub struct Article { pub author: String, pub content: String }

impl Summary for Article {
    fn summarize_author(&self) -> String { self.author.clone() }
    fn summarize(&self) -> String {
        format!("{}: {}", self.author, &self.content[..50.min(self.content.len())])
    }
}
```

### Generic functions and trait bounds
```rust
// impl Trait (preferred for simple cases)
fn notify(item: &impl Summary) {
    println!("Breaking: {}", item.summarize());
}

// Trait bound syntax (required for multiple params sharing a bound)
fn compare_summaries<T: Summary>(a: &T, b: &T) -> String {
    format!("{} vs {}", a.summarize(), b.summarize())
}

// Where clause (for complex bounds)
fn process<T>(item: T) -> String
where
    T: Summary + std::fmt::Debug,
{
    format!("{item:?}: {}", item.summarize())
}
```

### Newtype pattern for type safety
```rust
struct Meters(f64);
struct Kilograms(f64);

// Prevents accidentally passing Meters where Kilograms is expected
fn calculate_bmi(weight: Kilograms, height: Meters) -> f64 {
    weight.0 / (height.0 * height.0)
}
```

### Builder pattern
```rust
#[derive(Default)]
pub struct RequestBuilder {
    url: String,
    timeout_ms: u64,
    headers: Vec<(String, String)>,
}

impl RequestBuilder {
    pub fn new(url: impl Into<String>) -> Self {
        Self { url: url.into(), timeout_ms: 5000, ..Default::default() }
    }
    pub fn timeout(mut self, ms: u64) -> Self { self.timeout_ms = ms; self }
    pub fn header(mut self, k: impl Into<String>, v: impl Into<String>) -> Self {
        self.headers.push((k.into(), v.into())); self
    }
    pub fn build(self) -> Request { Request { url: self.url, timeout_ms: self.timeout_ms, headers: self.headers } }
}
```

---

## Smart Pointers

| Type | Use case |
|---|---|
| `Box<T>` | Heap allocation; recursive types; trait objects |
| `Rc<T>` | Multiple owners, single thread |
| `Arc<T>` | Multiple owners, multiple threads |
| `Cell<T>` | Interior mutability for `Copy` types |
| `RefCell<T>` | Interior mutability, runtime borrow checking |
| `Mutex<T>` | Exclusive access across threads |
| `RwLock<T>` | Many readers OR one writer across threads |

```rust
// Box: trait objects
fn make_animal(is_dog: bool) -> Box<dyn Animal> {
    if is_dog { Box::new(Dog) } else { Box::new(Cat) }
}

// Arc<Mutex<T>>: shared mutable state across threads
use std::sync::{Arc, Mutex};
let counter = Arc::new(Mutex::new(0u64));
let c = Arc::clone(&counter);
std::thread::spawn(move || { *c.lock().unwrap() += 1; });
```

---

## Concurrency

### Spawning threads and joining
```rust
use std::thread;

let handle = thread::spawn(|| {
    // work here
    42
});
let result = handle.join().expect("thread panicked");
```

### Channels (message passing)
```rust
use std::sync::mpsc;

let (tx, rx) = mpsc::channel::<String>();

thread::spawn(move || {
    tx.send("hello from thread".to_string()).unwrap();
});

let msg = rx.recv().unwrap();
println!("{msg}");
```

### Tokio async — concurrent tasks
```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // Run two futures concurrently
    let (a, b) = tokio::join!(
        fetch("https://example.com"),
        fetch("https://rust-lang.org"),
    );
}

async fn fetch(url: &str) -> String {
    sleep(Duration::from_millis(100)).await;
    format!("response from {url}")
}
```

### Blocking inside async
```rust
// Offload CPU-intensive work or blocking calls
let result = tokio::task::spawn_blocking(|| {
    heavy_computation()
}).await?;
```

---

## Iterators & Closures

### Prefer iterator chains
```rust
// ❌ Manual loop
let mut evens = vec![];
for x in 0..10 {
    if x % 2 == 0 { evens.push(x * x); }
}

// ✅ Iterator chain
let evens: Vec<_> = (0..10)
    .filter(|x| x % 2 == 0)
    .map(|x| x * x)
    .collect();
```

### Implementing Iterator
```rust
struct Counter { count: u32, max: u32 }

impl Counter {
    fn new(max: u32) -> Self { Self { count: 0, max } }
}

impl Iterator for Counter {
    type Item = u32;
    fn next(&mut self) -> Option<u32> {
        if self.count < self.max {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}

// Now all Iterator adapters work for free:
let sum: u32 = Counter::new(5).zip(Counter::new(5).skip(1)).map(|(a, b)| a * b).sum();
```

### Common iterator patterns
```rust
// flat_map
let words: Vec<&str> = ["hello world", "foo bar"]
    .iter()
    .flat_map(|s| s.split(' '))
    .collect();

// partition
let (evens, odds): (Vec<i32>, Vec<i32>) = (1..=10).partition(|x| x % 2 == 0);

// fold (general accumulation)
let product: i32 = (1..=5).fold(1, |acc, x| acc * x);

// chain
let combined: Vec<i32> = [1, 2].iter().chain([3, 4].iter()).copied().collect();
```

---

## Structs & Enums

### Struct update syntax
```rust
let base = Config::default();
let custom = Config { timeout: 30, ..base };
```

### Enums with data
```rust
#[derive(Debug)]
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    Color(u8, u8, u8),
}

fn handle(msg: Message) {
    match msg {
        Message::Quit => println!("quit"),
        Message::Move { x, y } => println!("move to {x},{y}"),
        Message::Write(text) => println!("write: {text}"),
        Message::Color(r, g, b) => println!("color #{r:02x}{g:02x}{b:02x}"),
    }
}
```

### Pattern matching tips
```rust
// if let for single arm
if let Some(value) = option { use(value) }

// while let
while let Some(top) = stack.pop() { process(top) }

// @ bindings
match number {
    n @ 1..=12 => println!("month {n}"),
    n @ 13..=19 => println!("teen {n}"),
    n => println!("other: {n}"),
}

// Destructuring in function params
fn print_point(&(x, y): &(i32, i32)) {
    println!("({x}, {y})");
}
```

---

## Modules & Visibility

```rust
// src/lib.rs
mod animals;           // loads src/animals.rs or src/animals/mod.rs
pub use animals::Dog;  // re-export for cleaner public API

// src/animals.rs
pub struct Dog { pub name: String }
pub(crate) struct InternalCat;  // visible only within crate
```

Visibility keywords:
- `pub` — public to everyone
- `pub(crate)` — visible within the crate
- `pub(super)` — visible to parent module
- (no modifier) — private to current module

---

## Testing

```rust
// Unit tests live in same file, under a test module
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    #[should_panic(expected = "overflow")]
    fn test_overflow() {
        overflow_fn();
    }

    #[test]
    fn test_result() -> Result<(), String> {
        let val = parse_number("42").map_err(|e| e.to_string())?;
        assert_eq!(val, 42);
        Ok(())
    }
}

// Integration tests: tests/integration_test.rs
// (only has access to public API of the crate)
```

Run with:
```sh
cargo test                          # all tests
cargo test test_add                 # filter by name
cargo test -- --nocapture           # show println output
cargo test -- --test-threads=1      # sequential
```

---

## Common Crates

| Purpose | Crate | Notes |
|---|---|---|
| Async runtime | `tokio` | Default choice; `features = ["full"]` for most apps |
| HTTP client | `reqwest` | Async; built on tokio |
| Serialization | `serde` + `serde_json` | `#[derive(Serialize, Deserialize)]` |
| Error handling (lib) | `thiserror` | Derive macro for error enums |
| Error handling (app) | `anyhow` | Ergonomic propagation |
| CLI args | `clap` | `#[derive(Parser)]` |
| Logging | `tracing` | Structured; replaces `log` |
| Parallelism | `rayon` | Data-parallel iterators |
| Date/time | `chrono` or `time` | `chrono` is more common |
| UUID | `uuid` | `features = ["v4"]` |
| Config | `config` or `figment` | Multi-source config |
| Database (async) | `sqlx` | Compile-time SQL checking |
| Regex | `regex` | Lazy static init with `once_cell` |

Standard Cargo.toml pattern:
```toml
[dependencies]
anyhow     = "1"
serde      = { version = "1", features = ["derive"] }
serde_json = "1"
tokio      = { version = "1", features = ["full"] }
tracing    = "0.1"

[dev-dependencies]
tokio-test = "0.4"
```
