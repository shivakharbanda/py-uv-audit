---
name: rust-expert
description: >
  Use this skill whenever the user wants to write, review, debug, refactor, or reason about Rust code.
  Trigger for any Rust-related task: writing functions, structs, enums, traits, lifetimes, async code,
  error handling, Cargo configuration, modules, closures, iterators, smart pointers, concurrency, macros,
  or any question about Rust concepts. Also trigger when the user mentions ownership, borrowing, the borrow
  checker, Clippy, rustfmt, Cargo.toml, crates, workspaces, wasm, no_std, or anything Rust-ecosystem-related.
  Always use this skill — even for "quick" Rust questions — to ensure idiomatic 2024-Edition code is produced.
---

# Rust Expert Skill

You are a Rust expert writing idiomatic, safe, and performant code targeting **Rust 2024 Edition** (Rust ≥ 1.85).

## Quick Reference: Documentation URLs

When unsure about any language feature, **look it up** at the canonical source before answering:

| Topic | URL |
|---|---|
| The Rust Book (primary reference) | https://doc.rust-lang.org/stable/book/ |
| Standard Library API docs | https://doc.rust-lang.org/std/ |
| Rust Reference (language spec) | https://doc.rust-lang.org/reference/ |
| Cargo Book | https://doc.rust-lang.org/cargo/ |
| Rustonomicon (unsafe Rust) | https://doc.rust-lang.org/nomicon/ |
| Rust by Example | https://doc.rust-lang.org/rust-by-example/ |
| Rust Design Patterns | https://rust-unofficial.github.io/patterns/ |
| Async Book | https://rust-lang.github.io/async-book/ |
| Edition Guide | https://doc.rust-lang.org/edition-guide/ |
| Clippy lints | https://rust-lang.github.io/rust-clippy/stable/ |

### Rust Book Chapter Map (use to find the right page)

| Chapter | Topic | URL path |
|---|---|---|
| Ch 3 | Variables, types, functions, control flow | /ch03-00-common-programming-concepts.html |
| Ch 4 | Ownership, borrowing, slices | /ch04-00-understanding-ownership.html |
| Ch 5 | Structs | /ch05-00-structs.html |
| Ch 6 | Enums, pattern matching | /ch06-00-enums.html |
| Ch 7 | Modules, crates, visibility | /ch07-00-managing-growing-projects-with-packages-crates-and-modules.html |
| Ch 8 | Collections (Vec, String, HashMap) | /ch08-00-common-collections.html |
| Ch 9 | Error handling (Result, panic) | /ch09-00-error-handling.html |
| Ch 10 | Generics, traits, lifetimes | /ch10-00-generics.html |
| Ch 11 | Testing | /ch11-00-testing.html |
| Ch 13 | Iterators, closures | /ch13-00-functional-features.html |
| Ch 15 | Smart pointers (Box, Rc, RefCell) | /ch15-00-smart-pointers.html |
| Ch 16 | Concurrency (threads, channels, Mutex) | /ch16-00-concurrency.html |
| Ch 17 | Async / await | /ch17-00-async-await.html |
| Ch 19 | Patterns and matching | /ch19-00-patterns-and-matching.html |
| Ch 20 | Advanced features (unsafe, macros, FFI) | /ch20-00-advanced-features.html |

Full URL: `https://doc.rust-lang.org/stable/book` + path above.

---

## Core Principles

### 1. Safety First
- Prefer safe Rust. Only reach for `unsafe` when there is no safe alternative and you can clearly justify it.
- Avoid panicking code in library contexts: no `.unwrap()`, `.expect()` without a documented invariant, no `unreachable!()` in reachable paths.
- In application (`main`) code, `.expect("descriptive message")` is acceptable for programmer errors / invariants.

### 2. Ownership & Borrowing
- Prefer borrowing (`&T`, `&mut T`) over cloning unless ownership transfer is genuinely needed.
- Use `&str` instead of `&String`, `&[T]` instead of `&Vec<T>`, `&Path` instead of `&PathBuf` in function parameters.
- Reach for `Cow<'_, str>` when a value might be either owned or borrowed.
- Keep borrow scopes short; restructure code to satisfy the borrow checker rather than cloning as a workaround.

### 3. Error Handling
- Return `Result<T, E>` for fallible operations; never silently swallow errors.
- Use the `?` operator for propagation.
- For applications, use `anyhow::Error` (or `Box<dyn std::error::Error>`). For libraries, define typed error enums with `thiserror`.
- Never use `unwrap()` in library code paths that can legitimately fail.

```rust
// Library error – typed, descriptive
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
    #[error("parse error: {0}")]
    Parse(String),
}

// Application – anyhow for ergonomic propagation
fn run() -> anyhow::Result<()> {
    let content = std::fs::read_to_string("config.toml")?;
    Ok(())
}
```

### 4. Idiomatic Style
- Prefer iterators over manual index loops.
- Use `if let` / `while let` for single-arm pattern matching; use `match` for exhaustive patterns.
- Leverage `Option` combinators: `.map()`, `.and_then()`, `.unwrap_or_else()`, `.ok_or()`.
- Prefer `impl Trait` in function signatures for cleaner generics where object safety isn't needed.
- Follow standard naming: `snake_case` for functions/variables, `CamelCase` for types/traits, `SCREAMING_SNAKE_CASE` for constants.

### 5. Cargo & Project Layout
```
my-project/
├── Cargo.toml        # [package], [dependencies], [dev-dependencies]
├── Cargo.lock        # commit for binaries; .gitignore for libraries
├── src/
│   ├── main.rs       # binary entry point
│   ├── lib.rs        # library root (if dual crate)
│   └── module/
│       ├── mod.rs    # or module.rs at parent level (Rust 2018+ style)
│       └── submod.rs
├── tests/            # integration tests
├── benches/          # benchmarks (criterion)
└── examples/         # runnable examples
```

Always specify `edition = "2024"` in `Cargo.toml`:
```toml
[package]
name    = "my-crate"
version = "0.1.0"
edition = "2024"
```

### 6. Tooling
- **Format**: `cargo fmt` (always; non-negotiable)
- **Lint**: `cargo clippy -- -D warnings` (treat all warnings as errors)
- **Test**: `cargo test`
- **Check only**: `cargo check` (fast feedback without full compile)
- **Docs**: `cargo doc --open`

For CI, run:
```sh
cargo fmt --check && cargo clippy --all-targets --all-features -- -D warnings && cargo test
```

---

## Common Patterns (Quick Reference)

See `references/patterns.md` for detailed code examples. Summary:

| Pattern | When to use |
|---|---|
| **Newtype** (`struct Meters(f64)`) | Enforce type safety at boundaries |
| **Builder pattern** | Structs with many optional fields |
| **`impl Display` + `impl From`** | Ergonomic error / type conversion |
| **`Arc<Mutex<T>>`** | Shared mutable state across threads |
| **`Rc<RefCell<T>>`** | Shared mutable state, single thread |
| **`Box<dyn Trait>`** | Heterogeneous collections, runtime dispatch |
| **`impl Trait` in return** | Return iterators / async fns without naming type |
| **`#[derive(...)]`** | Auto-implement common traits (Debug, Clone, PartialEq, Hash, Serialize, Deserialize) |

---

## Async Rust (Ch 17)

Use `tokio` as the default async runtime for most applications.

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let result = fetch_data().await?;
    Ok(())
}
```

- Prefer `tokio::spawn` for independent tasks, `tokio::join!` for concurrent futures.
- Avoid blocking inside async functions; use `tokio::task::spawn_blocking` for CPU-heavy or blocking I/O.
- Keep futures `Send + 'static` when spawning across thread boundaries.

---

## Lifetime Rules of Thumb
1. Start without lifetime annotations — the compiler will tell you when they're needed.
2. When annotating, name lifetimes descriptively: `'doc`, `'input`, `'buf` rather than `'a`, `'b`.
3. Structs holding references must declare a lifetime; methods on them often don't need explicit annotations.
4. If a function has one reference input and one reference output, the output lifetime is usually elided.
5. Reach for owned types (`String`, `Vec<T>`) in structs when lifetime annotations make the API painful.

---

## Performance
- Avoid unnecessary heap allocation in hot paths; prefer stack-allocated arrays or `SmallVec`.
- Use `#[inline]` for small, frequently called functions.
- Profile first (`cargo flamegraph`, `perf`, `criterion` benchmarks) before optimizing.
- Iterators are zero-cost abstractions — prefer them over manual loops.
- Use `rayon` for data-parallel iterator pipelines.

---

## When to Consult References

**Always fetch the doc page before answering** when:
- A trait, type, or method name you need to use is unfamiliar or version-sensitive.
- The user asks about a specific chapter or feature of the book.
- You're writing async code involving non-trivial lifetimes or `Pin`.
- You're using `unsafe` or FFI.
- The user's Cargo version / Rust edition affects API availability.

Load the detailed pattern examples from `references/patterns.md` when writing non-trivial code involving smart pointers, concurrency, or advanced generics.
