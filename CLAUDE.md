# uv-audit

Python vulnerability scanner for `uv`-managed projects, powered by a Rust core with PyO3 bindings.

## Project structure

| Path | Purpose |
|------|---------|
| `rust/` | Rust crate — library (`uv_audit`) + CLI binary |
| `rust/src/lib.rs` | All business logic + PyO3 Python bindings |
| `rust/src/main.rs` | Thin CLI wrapper (ANSI formatting, arg parsing) |

## Before editing any Rust code

**Always load the rust-expert skill first:**

```
/rust-expert
```

The skill is at `.claude/skills/rust-expert/SKILL.md`. It covers Rust 2024 edition idioms, ownership patterns, error handling with `anyhow`/`thiserror`, PyO3 bindings, and Clippy/fmt requirements.

Do not modify `rust/src/lib.rs` or `rust/src/main.rs` without the skill loaded.

## Building the CLI

```sh
cd rust
cargo build                    # compile
cargo run -- --tree            # dependency tree
cargo run -- --suggest         # vuln report + fix suggestions
cargo run -- --tree --suggest  # all three
cargo run -- --pyproject /path/to/pyproject.toml --lockfile /path/to/uv.lock
```

## Python bindings

The Rust lib exposes 3 functions via PyO3:

```python
import uv_audit

tree: str = uv_audit.dependency_tree("pyproject.toml", "uv.lock")
report     = uv_audit.vulnerability_scan("pyproject.toml", "uv.lock")
# report.total_scanned: int
# report.vulnerabilities: list[PyVulnerabilityReport]

suggestions = uv_audit.fix_suggestions("pyproject.toml", "uv.lock")
# each: PyFixSuggestion with package, fix_version, bump_type, is_direct, ...
```

Build the Python wheel with `maturin` (setup coming next).
