# Phase 03 — axum service with /embed and /search

**Depends on:** [02](02-ollama-client.md) · **Next:** [04-ingest-qdrant](04-ingest-qdrant.md) · **PRD:** [phase 3](../prd.md#phase-3--axum-service-with-embed-and-search)

## Context brief (cold start)

`rag-core` has the chunker and the `Embedder` / `ChatModel` traits; `rag-clients` has the Ollama client.
This phase adds the `rag-service` binary: an HTTP API with in-memory search (Qdrant comes in phase 04)
and an OpenAPI spec that Dify will import in phase 07. You write the Rust; Claude writes failing tests.

## Objective

`rag-service` answers `GET /health`, `POST /embed`, `POST /search`, `GET /openapi.json` on port 3000.

## Concepts

- Routers, handlers, extractors (`Json`, `State`) — [axum docs](https://docs.rs/axum) · [axum examples](https://github.com/tokio-rs/axum/tree/main/examples)
- Shared state: `Arc` — [Book 16.3](https://doc.rust-lang.org/book/ch16-03-shared-state.html)
- `Send` + `Sync` — [Book 16.4](https://doc.rust-lang.org/book/ch16-04-extensible-concurrency-sync-and-send.html)
- Errors as HTTP responses: `IntoResponse` — [axum error handling](https://docs.rs/axum/latest/axum/error_handling/index.html)
- Iterators for ranking — [Book 13.2](https://doc.rust-lang.org/book/ch13-02-iterators.html)
- Cosine similarity — [Qdrant: distance metrics](https://qdrant.tech/documentation/concepts/search/#metrics)
- OpenAPI — [OpenAPI 3.1 spec](https://spec.openapis.org/oas/v3.1.0)

## Steps

### 1. Hello service
```bash
cargo new crates/rag-service
cargo add -p rag-service axum tokio --features tokio/full
cargo run -p rag-service
```
Bind to `0.0.0.0:3000` so Docker containers can reach it later.
**Done check:** `curl -s localhost:3000/health` returns `ok` (or `{"status":"ok"}`).

### 2. Router tests first
The tests need test-only dependencies (Claude confirms the names on docs.rs and in axum's `testing`
example first):
```bash
cargo add -p rag-service --dev tower --features util
cargo add -p rag-service --dev http-body-util
```
Ask Claude: *"write the failing router tests for phase 03"*. They call the router directly without a
network (axum's testing example uses `tower::ServiceExt::oneshot`) with a fake `Embedder`, and cover:
`/embed` 200 with vectors; `/embed` with empty input → 400; embedder failure → 502 with a JSON error;
`/search` returns results sorted by score, at most `top_k`.

Hint: for the fake embedder to plug in, the state must hold the trait, not the Ollama type. Decide between
a generic and `Arc<dyn Embedder>` and be ready to say why.

### 3. /embed and /search
`/search` embeds the query and ranks an in-memory list of chunks (load one Markdown file at startup).
The response shape is fixed here and used by every later phase:

- request: `{"query": "...", "top_k": 5}`
- response: `{"results": [{"text": "...", "source": "ch04-02-references-and-borrowing.md", "heading": "...", "score": 0.83}]}`
- `source` is the **file name only** (no directory), so it matches `expected_sources` in phase 05.

```bash
cargo test -p rag-service
curl -s localhost:3000/embed -H 'content-type: application/json' -d '{"input":["hi"]}' | jq '.embeddings[0] | length'
curl -s localhost:3000/search -H 'content-type: application/json' -d '{"query":"ownership","top_k":3}' \
  | jq '(.results | length) == 3 and ([.results[].score] == ([.results[].score] | sort | reverse))'
```
**Done check:** tests pass; the first curl prints the dimension; the second prints `true`.

### 4. OpenAPI spec
Either hand-write `openapi.yaml` and serve it, or generate it from code with a crate (ask Claude to check
the options on docs.rs first). Each operation needs an `operationId`: Dify uses it as the tool name. Set
`servers: [{url: "http://host.docker.internal:3000"}]`: Dify calls the URL in `servers`, and from inside
Docker `localhost` is the container itself.
```bash
curl -s localhost:3000/openapi.json | jq '.paths | keys'
```
**Done check:** lists `/embed`, `/health`, `/search`. Optional now, required in phase 07: Dify's custom-tool
import accepts it.

## Phase done check
```bash
cargo test --workspace && cargo clippy --workspace -- -D warnings
```

## Stretch

- Request logging with tracing — [tokio tracing](https://tokio.rs/tokio/topics/tracing).
- Graceful shutdown on Ctrl-C — see the axum `graceful-shutdown` example.
