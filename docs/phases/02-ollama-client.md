# Phase 02 — Async HTTP client for Ollama

**Depends on:** [01](01-rust-basics-cli.md) · **Next:** [03-axum-service](03-axum-service.md) · **PRD:** [phase 2](../prd.md#phase-2--async-http-client-for-ollama)

## Context brief (cold start)

The workspace has `rag-core` (chunker) and `rag-cli`. Ollama runs on `localhost:11434` with the models in
`.env`. This phase adds traits for models in `rag-core` and a new crate `rag-clients` with the Ollama
implementation. You write the Rust; Claude writes failing tests and gives hints.

## Objective

`rag-cli embed "text"` prints the vector dimension, and `rag-cli chat "question"` prints an answer.

## Concepts

- async/await and the runtime — [Book ch. 17](https://doc.rust-lang.org/book/ch17-00-async-await.html) · [tokio tutorial](https://tokio.rs/tokio/tutorial)
- Traits — [Book 10.2](https://doc.rust-lang.org/book/ch10-02-traits.html)
- Your own error type and `From` for `?` — [Book 9.2](https://doc.rust-lang.org/book/ch09-02-recoverable-errors-with-result.html) · [std::error::Error](https://doc.rust-lang.org/std/error/trait.Error.html) · [Rust by Example: wrapping errors](https://doc.rust-lang.org/rust-by-example/error/multiple_error_types/wrap_error.html)
- JSON with serde — [serde.rs](https://serde.rs/) · [derive](https://serde.rs/derive.html)
- HTTP client — [reqwest](https://docs.rs/reqwest)
- Ollama API — [/api/embed](https://docs.ollama.com/api/embed) · [/api/chat](https://docs.ollama.com/api/chat)
- Timeouts — [tokio::time::timeout](https://docs.rs/tokio/latest/tokio/time/fn.timeout.html)

## Steps

### 1. Traits in `rag-core`
Define an `Embedder` (texts in → vectors out) and a `ChatModel` (messages in → text out). `rag-core`
stays free of HTTP. Hint: think about whether `embed` takes one text or many — Ollama accepts an array,
and phase 4 will embed hundreds of chunks.

Async functions in traits: read the Book ch. 17 section on traits and futures. **Look ahead:** in
phases 03 and 05 you'll want to hold "some embedder" or "some chat model" behind `dyn` (a fake in tests,
Ollama or Claude at runtime). A trait with a native `async fn` is not dyn-compatible — read
[Reference: dyn compatibility](https://doc.rust-lang.org/reference/items/traits.html#dyn-compatibility)
and [async-trait](https://docs.rs/async-trait), then decide now which way you go. Ask Claude if you get
stuck on `Send` bounds.

**Done check:** `cargo build -p rag-core`.

### 2. Create `rag-clients` and add dependencies
```bash
cargo new --lib crates/rag-clients
cargo add -p rag-clients tokio --features full
cargo add -p rag-clients reqwest --features json
cargo add -p rag-clients serde --features derive
cargo add -p rag-clients serde_json
```
**Done check:** `cargo build -p rag-clients`.

### 3. Ollama implementation (test first)
Ask Claude: *"write the failing tests for the Ollama client, phase 02"*. They use a local fake HTTP server
(Claude checks the crate on docs.rs first) and cover: the request body sent to `/api/embed`; parsing
`embeddings`; HTTP 500 → your error variant; a slow server → a timeout error; a wrong-shaped body → a
decode error.

Hint: model the request and response as structs with only the fields you use.
```bash
cargo test -p rag-clients
```
**Done check:** all tests pass without Ollama running.

### 4. Wire into the CLI
The CLI reads `OLLAMA_URL`, `EMBED_MODEL`, `CHAT_MODEL` from the environment
([std::env::var](https://doc.rust-lang.org/std/env/fn.var.html)). Load `.env` into your shell with
`set -a; source .env; set +a`, or use a dotenv crate (ask Claude to check it on docs.rs first).
```bash
cargo add -p rag-cli tokio --features full
cargo add -p rag-cli --path crates/rag-clients
set -a; source .env; set +a
cargo run -p rag-cli -- embed "hello"
cargo run -p rag-cli -- chat "what is ownership in Rust?"
```
**Done check:** the first prints the same dimension as phase 00 step 2; the second prints an answer.
Then run `OLLAMA_URL=http://localhost:1 cargo run -p rag-cli -- embed "hello"` (nothing listens on
port 1): you get a readable error and a non-zero exit, not a panic.

## Phase done check
```bash
cargo test --workspace && cargo clippy --workspace -- -D warnings
```

## Stretch

- Stream chat tokens (`"stream": true` in `/api/chat`) and print them as they arrive.
- A second `ChatModel` for Claude — [Messages API](https://docs.anthropic.com/en/api/messages). Key from
  `ANTHROPIC_API_KEY` in `.env`; never in code.
