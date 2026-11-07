# Phase 01 — Rust basics: Rustlings + a chunking CLI

**Depends on:** [00](00-env-setup.md) · **Next:** [02-ollama-client](02-ollama-client.md) · **PRD:** [phase 1](../prd.md#phase-1--rust-basics-rustlings--small-cli)

## Context brief (cold start)

Rust is installed, services run. No `crates/` yet. This phase creates the Cargo workspace and two crates:
`rag-core` (pure logic, no I/O) and `rag-cli` (binary). **You write all the Rust.** Claude writes the
failing tests for the chunker when you ask, and gives hints.

Rustlings is self-paced and may take several sessions on its own. This file's session is the CLI part:
start it once Rustlings is done through the error-handling exercises.

## Objective

`cargo run -p rag-cli -- chunk <file.md>` prints numbered chunks of a Markdown file.

## Concepts

- Ownership, borrowing, slices — [Book 4.1](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html) · [4.2](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html) · [4.3](https://doc.rust-lang.org/book/ch04-03-slices.html)
- Structs, enums, `match` — [Book ch. 5](https://doc.rust-lang.org/book/ch05-00-structs.html) · [ch. 6](https://doc.rust-lang.org/book/ch06-00-enums.html)
- `Result` and `?` — [Book 9.2](https://doc.rust-lang.org/book/ch09-02-recoverable-errors-with-result.html)
- Modules and crates — [Book ch. 7](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html)
- Tests — [Book 11.1](https://doc.rust-lang.org/book/ch11-01-writing-tests.html) · [11.3 organization](https://doc.rust-lang.org/book/ch11-03-test-organization.html)
- Workspaces — [Book 14.3](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html) · [Cargo Book: workspaces](https://doc.rust-lang.org/cargo/reference/workspaces.html)
- Strings are UTF-8; byte index ≠ char index — [Book 8.2](https://doc.rust-lang.org/book/ch08-02-strings.html)

## Steps

### 1. Rustlings (outside this repo)
Docs: [Rustlings](https://rustlings.rust-lang.org/)
```bash
cargo install rustlings
cd ~ && rustlings init && cd rustlings && rustlings
```
**Done check:** the exercises through `13_error_handling` pass.

### 2. Create the workspace
Read [Book 14.3](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html) first, then write a root
`Cargo.toml` with a `[workspace]` section yourself.
```bash
cargo new --lib crates/rag-core
cargo new crates/rag-cli
cargo build --workspace
```
**Done check:** `cargo build --workspace` succeeds and creates one `target/` at the repo root.

### 3. Chunker in `rag-core` (test first)
Ask Claude: *"write the failing chunker tests for phase 01"*. They will cover: empty input → no chunks;
short text → one chunk; long text → chunks no longer than the max size; overlap between neighbours;
a split at Markdown headings; multi-byte UTF-8 (no panic, no broken characters).

Hint: decide what a chunk is (a struct with text plus where it came from) before you decide how to cut.
Ask yourself whether the chunker needs to own the text or can borrow it.
```bash
cargo test -p rag-core
```
**Done check:** all chunker tests pass.

### 4. CLI in `rag-cli`
Docs: [Book ch. 12 (an I/O project)](https://doc.rust-lang.org/book/ch12-00-an-io-project.html) ·
[std::env::args](https://doc.rust-lang.org/std/env/fn.args.html) · [std::fs::read_to_string](https://doc.rust-lang.org/std/fs/fn.read_to_string.html)

The CLI only reads the file, calls the chunker and prints. A missing file must print an error and exit
non-zero, not panic.
```bash
cargo run -p rag-cli -- chunk CLAUDE.md
cargo run -p rag-cli -- chunk does-not-exist.md; echo "exit=$?"
```
**Done check:** numbered chunks for the first; an error message and `exit=1` (non-zero) for the second.

## Phase done check
```bash
cargo test --workspace && cargo clippy --workspace -- -D warnings && cargo fmt --check
```

## Stretch

- Argument parsing with a crate (ask Claude to check the current one on docs.rs first).
- Make chunks borrow `&str` slices from the input, and explain the lifetime in a comment.
