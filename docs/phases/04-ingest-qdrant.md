# Phase 04 — Ingest pipeline + Qdrant

**Depends on:** [03](03-axum-service.md) · **Next:** [05-rag-ask-eval](05-rag-ask-eval.md) · **PRD:** [phase 4](../prd.md#phase-4--ingest-pipeline--qdrant)

## Context brief (cold start)

`rag-service` has `/embed` and an in-memory `/search`. Qdrant runs in Docker (REST 6333, gRPC 6334). This
phase adds a `VectorStore` trait, a Qdrant implementation, an ingest pipeline (read → chunk → embed →
upsert) and moves `/search` to Qdrant. The corpus is The Rust Book's Markdown. You write the Rust.

This phase is two sessions: **A** = steps 1–2 (VectorStore + Qdrant), **B** = steps 3–4 (ingest + endpoints).

## Objective

`rag-cli ingest data/rust-book/src` fills a Qdrant collection, re-running it adds no duplicates, and
`/search` returns chunks from Qdrant.

## Concepts

- Collections, points, payload — [Qdrant collections](https://qdrant.tech/documentation/concepts/collections/) · [points](https://qdrant.tech/documentation/concepts/points/) · [payload](https://qdrant.tech/documentation/concepts/payload/)
- Rust client (gRPC, port **6334**) — [qdrant-client](https://docs.rs/qdrant-client)
- Bounded concurrency — [tokio::sync::Semaphore](https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html) · [futures StreamExt::buffer_unordered](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.buffer_unordered)
- Lifetimes (if your chunker borrows) — [Book 10.3](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html)
- Walking a directory — [std::fs::read_dir](https://doc.rust-lang.org/std/fs/fn.read_dir.html)

## Steps

### 1. Get the corpus
```bash
git clone --depth 1 https://github.com/rust-lang/book data/rust-book
ls data/rust-book/src/*.md | wc -l
```
**Done check:** about 100 files. `data/` is gitignored.

### 2. `VectorStore` trait + Qdrant implementation (test first)
```bash
cargo add -p rag-clients qdrant-client
```
Ask Claude: *"write the failing tests for the VectorStore, phase 04"*. Unit tests use an in-memory fake;
one `#[ignore]` integration test hits the real Qdrant (`cargo test -- --ignored`).

Hints: the collection's vector size must equal `EMBED_DIM`. Give each point a **deterministic id** (from
source path + chunk index) so upserting twice overwrites instead of duplicating. Qdrant ids are an
unsigned int or a UUID. Store `source` (file name only, as defined in phase 03), `heading`, `text` in the payload.

**Done check:** `cargo test -p rag-clients` and `cargo test -p rag-clients -- --ignored` pass.

### 3. Ingest pipeline
Read files → chunk → embed in batches → upsert. Limit how many embed calls run at once (start with 4).
One bad file must not stop the whole run: log it and continue, then report a count at the end.
```bash
cargo add -p rag-core futures   # or wherever the pipeline lives; for StreamExt::buffer_unordered
set -a; source .env; set +a
cargo run -p rag-cli -- ingest data/rust-book/src
curl -s localhost:6333/collections/$QDRANT_COLLECTION | jq '.result.points_count'
cargo run -p rag-cli -- ingest data/rust-book/src
curl -s localhost:6333/collections/$QDRANT_COLLECTION | jq '.result.points_count'
```
**Done check:** both counts are equal and greater than 0.

### 4. `/search` and `/ingest` on Qdrant
`/ingest` takes raw text so n8n can call it: request `{"source": "notes.md", "text": "..."}`, response
`{"chunks": <n>}`. It runs the same chunk → embed → upsert path as the CLI.
```bash
curl -s localhost:3000/search -H 'content-type: application/json' -d '{"query":"what is a borrow","top_k":3}' | jq '.results[].source'
curl -s localhost:3000/ingest -H 'content-type: application/json' -d '{"source":"notes.md","text":"Zebras borrow nothing."}' | jq '.chunks'
curl -s localhost:3000/search -H 'content-type: application/json' -d '{"query":"zebras","top_k":1}' | jq -r '.results[0].source'
```
**Done check:** the first lists `ch04-02-references-and-borrowing.md` at least once; the second prints
`1`; the third prints `notes.md`. Also open the Qdrant dashboard at `http://localhost:6333/dashboard`
and find one of your points.

## Phase done check
```bash
cargo test --workspace && cargo clippy --workspace -- -D warnings
```

## Stretch

- Heading-aware chunker vs fixed-size: keep both, choose by flag. Phase 05 measures which is better.
- `DELETE` by source (payload filter) so changing one file re-ingests only that file.
