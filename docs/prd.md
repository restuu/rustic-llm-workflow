# PRD: rustic-llm-workflow — learn Rust by building a RAG service for n8n and Dify

*Status: DRAFT, waiting for approval · Created 2026-11-06*

## Problem statement

A senior backend engineer who has never written Rust or used n8n, and has used Dify only a little, wants
to learn all three by building something real, not by doing tutorials alone. The cost of not doing it
this way: tutorial knowledge that never meets async I/O, real error paths, or a real integration.

## Evidence

- The user's own goal: learn Rust "the hard way" through one capstone (stated in the request).
- Everything else is an **assumption**, to be checked at each phase's done check. If a phase takes
  more than about three sessions, split it.

## Proposed solution

One Rust HTTP service, built up phase by phase: ingest → chunk → embed → store → retrieve → generate.
It runs locally against Ollama, with Claude as a hosted model to compare against. It is called from an
n8n workflow and a Dify workflow, and compared with Dify's built-in RAG on the same evaluation set.
Each phase teaches one group of Rust concepts and ends with a runnable done check.

## Key hypothesis

We believe that building a RAG service phase by phase will teach Rust's ownership, error handling,
traits, async and testing well enough for the user to write the service without copied solutions.
We'll know we're right when phase 5's evaluation passes (`cargo test` green, hit@5 ≥ 0.8 on the
evaluation set), and the user wrote all the Rust.

## What we're NOT building

- Production deployment, cloud hosting, Kubernetes: this is a local learning setup.
- A frontend: n8n and Dify are the user interfaces.
- Auth beyond one static API key: enough to learn Dify's custom-header auth, nothing more.
- Our own embedding model or vector index: Ollama and Qdrant do that.
- A fine-tuned model by default: phase 8 only runs if the decision point says so.

## Success metrics

| Metric | Target | How measured |
|---|---|---|
| Rust written by the user | 100% of `crates/` | Git history; Claude writes tests and hints only |
| Retrieval quality | hit@5 ≥ 0.8, MRR ≥ 0.6 | `eval` CLI over `eval/questions.jsonl` (phase 5) |
| Answer faithfulness | ≥ 80% judged "supported" | LLM-as-judge with Claude over the same set (phase 5) |
| Workflow integration | n8n and Dify both get an answer with sources | A workflow run in each tool (phases 6, 7) |
| Comparison written | One table: our RAG vs Dify built-in RAG | `docs/results.md` (phase 7) |

The thresholds are starting guesses. Adjust them after the first baseline run in phase 5 and log the
change in the decisions log.

## Open questions

- [ ] Corpus: the default is The Rust Book's Markdown source (MIT/Apache, about 100 files, on topic so
      you can sanity-check answers). Swap it if you have a corpus you care about more.
- [ ] Ollama models: one chat model and one embedding model, picked in phase 0 by checking the Ollama
      library for what fits your Mac's memory. The docs' embedding example is `embeddinggemma`.
- [ ] Claude model for comparison and judging: pick in phase 5 from docs.anthropic.com.

---

## Users and context

**Primary user:** you. You know backend systems (HTTP, databases, queues, testing), you're new to Rust
and n8n, and you have touched Dify.
**Trigger:** each session, you open the next `docs/phases/NN-*.md` and work through it.
**Success state:** the done check passes, and you can explain why your code compiles.

**Job to be done:** when I have a free session, I want one bounded step with a doc link, exact
commands and a done check, so I can make real progress in Rust without being handed the answer.

**Non-users:** anyone deploying this. It is not a product.

## How Claude works in this repo (see CLAUDE.md)

Learning mode: Claude explains concepts, links primary docs, gives hints and **failing tests**. Claude
does not write Rust implementations. Non-Rust glue (compose files, curl, workflow config) is fine.

The repo is meant to go public so anyone can fork it and take the same course. `master` holds only the
course (CLAUDE.md, `.gitignore`, this PRD, `docs/phases/`). Everything made while doing the phases (Rust
code, tests, `infra/`, eval set, workflow exports, results) is committed on a learner branch such as
`solutions`, never on `master`.

---

## Learning objectives

### Rust
| Area | You can… | Phase |
|---|---|---|
| Ownership & borrowing | explain move vs borrow; fix E0382/E0502 without cloning everything; choose `&str` vs `String` in a signature | 1, 4 |
| Error handling | use `Result` and `?`; define a domain error enum; map it to HTTP status codes | 1, 2, 3 |
| Traits & generics | define an `Embedder` / `ChatModel` trait with two implementations (Ollama, Claude); use trait objects vs generics on purpose | 2, 5 |
| Async / tokio | explain what `.await` does; share state with `Arc`; run bounded concurrency over many chunks | 2, 3, 4 |
| Testing | unit tests, integration tests in `tests/`, tests against a fake HTTP server, `cargo test` in a workspace | all |
| Tooling | `cargo` workspace, `clippy`, `rustfmt`, reading docs.rs | 1 → |

### LLM workflow tools
| Area | You can… | Phase |
|---|---|---|
| n8n | build a workflow with Webhook/Manual trigger → HTTP Request → output; give an AI Agent node your service as a tool | 6 |
| Dify | import an OpenAPI spec as a custom tool; call it from a workflow; build a built-in knowledge base; connect an external knowledge API | 7 |

### RAG
| Area | You can… | Phase |
|---|---|---|
| Chunking | compare fixed-size vs heading-aware chunking, with overlap, using eval numbers | 4, 5 |
| Embeddings | explain vector dimension and cosine similarity; call Ollama `/api/embed` in batches | 2, 3 |
| Vector search | create a Qdrant collection, upsert with payload, search with a filter | 4 |
| Reranking | add a rerank step (LLM-scored or a cross-encoder) and measure whether it helps | 5 (stretch) |
| Evaluation | build a 30–50 question set with expected sources; compute hit@k and MRR; LLM-as-judge for faithfulness | 5 |

### Training decision point (end of phase 7)
Fine-tune (phase 8) **only if** the evaluation shows a failure that retrieval can't fix:
- Answers have the right sources (hit@5 good) but the wrong **format or style**, even after prompt fixes, or
- The small local model keeps misusing **domain vocabulary** that is present in the retrieved context.

If retrieval is the problem (hit@5 low), fix chunking, embeddings or reranking first. Fine-tuning does
not fix missing knowledge. Write the decision with numbers in `docs/results.md`.

---

## Architecture (decided)

```
 n8n (docker) ──HTTP Request──┐
                              ├──► rag-service (Rust, axum, :3000, on the host)
 Dify (docker) ──custom tool──┘        │            │                 │
         └──external KB /retrieval─────┘            ▼                 ▼
                                             Ollama (native)   Qdrant (docker)
                                             chat + embeddings  vectors + payload
                                                    │
                                             Claude API (comparison, judging)
```

- The Rust service runs natively on the host (`cargo run`). Containers reach it at
  `http://host.docker.internal:3000`.
- **Ollama runs natively, not in Docker**: on a Mac, Docker can't use the Apple GPU (Metal), so models
  in a container run on the CPU only.
- `infra/docker-compose.yml` runs Qdrant and n8n. **Dify uses its own compose file** from its repo (it
  starts a dozen-plus services). Clone it into `infra/dify/`, which is gitignored, and run it from there.

### HTTP API (built up across phases)
| Endpoint | Phase | Purpose |
|---|---|---|
| `GET /health` | 3 | liveness |
| `POST /embed` | 3 | text(s) → vectors (thin wrapper over Ollama) |
| `POST /search` | 3 → 4 | query → top-k chunks (in memory in 3, Qdrant in 4) |
| `POST /ingest` | 4 | path or text → chunk → embed → upsert |
| `POST /ask` | 5 | question → retrieve → generate an answer with sources; `model=ollama\|claude` |
| `POST /retrieval` | 7 | Dify external knowledge API format |
| `GET /openapi.json` | 3 → | OpenAPI spec, which Dify imports |

### Workspace layout (the user creates it in phase 1–3)
```
crates/
  rag-core/     # chunking, traits (Embedder, ChatModel, VectorStore), domain types — no I/O
  rag-clients/  # Ollama, Claude, Qdrant implementations
  rag-service/  # axum binary
  rag-cli/      # ingest + eval CLI
```

---

## Phases

Each phase builds on the last. Full steps are in `docs/phases/NN-*.md`. Every phase has: objective,
concepts, doc links, commands, a minimal guideline, a done check and stretch goals. Crate versions are
never pinned here: `cargo add <crate>` takes the current one, and Claude checks docs.rs/Context7 at the
start of the phase.

| # | Phase | Depends | Delivers |
|---|---|---|---|
| 0 | Environment setup | – | rustup, Docker, Ollama models, Qdrant + n8n + Dify running |
| 1 | Rust basics: Rustlings + small CLI | 0 | `rag-cli chunk <file>` printing chunks |
| 2 | Async HTTP client for Ollama | 1 | `rag-cli embed "text"` and `rag-cli chat "q"` |
| 3 | axum service: /embed, /search | 2 | service with in-memory search + OpenAPI |
| 4 | Ingest pipeline + Qdrant | 3 | `/ingest` and `/search` backed by Qdrant |
| 5 | RAG /ask + evaluation set | 4 | `/ask` with sources; eval CLI with hit@k, MRR, judge |
| 6 | n8n workflow | 5 | n8n workflow + AI Agent calling the service |
| 7 | Dify workflow + comparison | 5 | Dify tool, external KB, built-in KB; `docs/results.md` |
| 8 | Optional: LoRA fine-tune (MLX) | 7 + decision | adapter; eval before/after |

Phases 6 and 7 both depend only on 5, and can run in either order.

### Phase 0 — Environment setup
- **Objective:** every tool runs and answers a health check.
- **Concepts:** rustup toolchains; Docker Compose; Ollama model pulls; how containers reach the host.
- **Docs:** [rustup install](https://www.rust-lang.org/tools/install) ·
  [Ollama quickstart](https://docs.ollama.com/quickstart) ·
  [Qdrant quickstart](https://qdrant.tech/documentation/quickstart/) ·
  [n8n Docker install](https://docs.n8n.io/deploy/host-n8n/install-options/install-with-docker) ·
  [Dify Docker Compose](https://docs.dify.ai/en/self-host/deploy/quick-start/docker-compose)
- **Commands:**
  ```bash
  curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
  rustup component add clippy rustfmt
  ollama pull <embedding-model>   # pick in this phase
  ollama pull <chat-model>        # pick in this phase
  docker compose -f infra/docker-compose.yml up -d
  git clone --branch "$(curl -s https://api.github.com/repos/langgenius/dify/releases/latest | jq -r .tag_name)" https://github.com/langgenius/dify.git infra/dify
  cd infra/dify/docker && cp .env.example .env && docker compose up -d
  ```
- **Guideline:** Claude writes `infra/docker-compose.yml` (Qdrant + n8n, with
  `extra_hosts: host.docker.internal:host-gateway`) after checking the image tags.
- **Done check:** `cargo --version`; `curl localhost:11434/api/embed -d '{"model":"<embedding-model>","input":"hi"}'`
  returns `embeddings`; `curl localhost:6333/collections` returns JSON; n8n at :5678 and Dify at :80 load.
- **Stretch:** a `make check-env` or shell script that runs all the done checks.

### Phase 1 — Rust basics: Rustlings + small CLI
- **Objective:** write a CLI that reads a Markdown file and prints chunks.
- **Concepts:** ownership, borrowing, slices, `String` vs `&str`, structs, enums, `match`, `Result`/`?`,
  modules, unit tests, Cargo workspaces.
- **Docs:** [The Book ch. 1–11](https://doc.rust-lang.org/book/) ·
  [Rustlings](https://rustlings.rust-lang.org/) ·
  [Cargo workspaces](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html) ·
  [std::fs](https://doc.rust-lang.org/std/fs/)
- **Commands:** `cargo install rustlings && rustlings init && cd rustlings && rustlings`;
  `cargo new --lib crates/rag-core`; `cargo new crates/rag-cli`; `cargo test`
- **Guideline:** finish Rustlings up to error handling first. Chunking is a pure function in `rag-core`
  (text in, chunks out, no I/O). The CLI only reads the file and prints. Claude provides failing tests for
  the chunker: empty input, one short paragraph, overlap, split on headings, multi-byte UTF-8.
- **Done check:** `cargo test -p rag-core` is green; `cargo run -p rag-cli -- chunk CLAUDE.md` prints
  numbered chunks; `cargo clippy -- -D warnings` is clean.
- **Stretch:** argument parsing with a crate (checked first); a chunker that borrows (`&str` slices)
  instead of allocating.

### Phase 2 — Async HTTP client for Ollama
- **Objective:** call Ollama's embed and chat APIs from Rust.
- **Concepts:** async/`.await`, the tokio runtime, serde (de)serialization, custom error enums,
  `From` for `?`, traits (`Embedder`, `ChatModel`), testing against a fake HTTP server.
- **Docs:** [Book ch. 17 async](https://doc.rust-lang.org/book/ch17-00-async-await.html) ·
  [tokio tutorial](https://tokio.rs/tokio/tutorial) · [reqwest](https://docs.rs/reqwest) ·
  [serde](https://serde.rs/) · [Ollama /api/embed](https://docs.ollama.com/api/embed) ·
  [Ollama /api/chat](https://docs.ollama.com/api/chat)
- **Commands:** `cargo new --lib crates/rag-clients`; `cargo add -p rag-clients tokio --features full`;
  `cargo add -p rag-clients reqwest --features json`; `cargo add -p rag-clients serde --features derive`;
  `cargo add -p rag-clients serde_json`
- **Guideline:** the traits live in `rag-core`, the implementations in `rag-clients`. Batch inputs
  (`/api/embed` accepts an array). Claude provides failing integration tests that use a fake server and
  check: the request body shape, parsing the vectors, an HTTP 500 → your error type, a timeout.
- **Done check:** `cargo test -p rag-clients` green; `cargo run -p rag-cli -- embed "hello"` prints the
  dimension; `cargo run -p rag-cli -- chat "what is ownership?"` prints an answer.
- **Stretch:** stream chat tokens; a second `ChatModel` implementation for Claude
  ([Messages API](https://docs.anthropic.com/en/api/messages)).

### Phase 3 — axum service with /embed and /search
- **Objective:** an HTTP service with shared state, in-memory vector search, and an OpenAPI spec.
- **Concepts:** axum routers, extractors, `State` with `Arc`, `IntoResponse` for errors, cosine
  similarity, `Send + Sync`, integration tests that call the router without a network.
- **Docs:** [axum](https://docs.rs/axum) · [axum examples](https://github.com/tokio-rs/axum/tree/main/examples) ·
  [Book ch. 16 Arc/Mutex](https://doc.rust-lang.org/book/ch16-03-shared-state.html) ·
  [OpenAPI 3.1 spec](https://spec.openapis.org/oas/v3.1.0)
- **Commands:** `cargo new crates/rag-service`; `cargo add -p rag-service axum`; `cargo run -p rag-service`
- **Guideline:** `/search` embeds the query and ranks an in-memory `Vec` of chunks. Errors map to
  400/502/500 with a JSON body. The OpenAPI spec can be hand-written YAML or generated by a crate
  (Claude checks the options first). Claude provides failing router tests for each endpoint.
- **Done check:**
  `curl -s localhost:3000/embed -H 'content-type: application/json' -d '{"input":["hi"]}' | jq '.embeddings[0] | length'`
  prints the dimension; `/search` returns ranked results; `curl localhost:3000/openapi.json` is valid
  (paste it into Dify's tool import, which must accept it).
- **Stretch:** request tracing; graceful shutdown.

### Phase 4 — Ingest pipeline + Qdrant
- **Objective:** ingest the corpus into Qdrant and search it.
- **Concepts:** bounded concurrency (`buffer_unordered` or a semaphore), backpressure, idempotent ids
  (hash of source + chunk index), lifetimes in the chunker, payload design.
- **Docs:** [Qdrant collections](https://qdrant.tech/documentation/concepts/collections/) ·
  [Qdrant points](https://qdrant.tech/documentation/concepts/points/) ·
  [qdrant-client](https://docs.rs/qdrant-client) · [tokio::sync::Semaphore](https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html)
- **Commands:** `cargo add qdrant-client -p rag-clients`;
  `git clone --depth 1 https://github.com/rust-lang/book data/rust-book`;
  `cargo run -p rag-cli -- ingest data/rust-book/src`
- **Guideline:** the Rust client talks gRPC on **6334**; the REST API and dashboard are on 6333. The
  collection's vector size must match your embedding model. Re-running ingest must not duplicate points.
- **Done check:** `curl localhost:6333/collections/<name>` shows `points_count` > 0 and the same count
  after a second ingest; `/search "what is a borrow"` returns a chunk from the ownership chapter.
- **Stretch:** a heading-aware chunker vs fixed-size, compared in phase 5; delete by source.

### Phase 5 — RAG /ask + evaluation set
- **Objective:** generate grounded answers with sources, and measure quality.
- **Concepts:** prompt assembly, trait objects (`dyn ChatModel`; async traits need a workaround to be
  `dyn`, see phase 02) to switch Ollama ↔ Claude,
  timeouts, eval metrics (hit@k, MRR), LLM-as-judge.
- **Docs:** [Ollama /api/chat](https://docs.ollama.com/api/chat) ·
  [Claude Messages API](https://docs.anthropic.com/en/api/messages) ·
  [Book ch. 18 trait objects](https://doc.rust-lang.org/book/ch18-02-trait-objects.html)
- **Commands:** `cargo run -p rag-cli -- eval eval/questions.jsonl --k 5`;
  `curl -s localhost:3000/ask -H 'content-type: application/json' -d '{"question":"...","model":"ollama"}'`
- **Guideline:** write 30–50 questions yourself with the expected source file for each. Claude may
  draft candidates, and you check them. Record a baseline before you change anything. `ANTHROPIC_API_KEY`
  goes in `.env`, never in code.
- **Done check:** `/ask` returns `answer` + `sources`; the eval prints hit@5, MRR and judge score for
  both models; the results are in `docs/results.md`.
- **Stretch:** reranking; hybrid search (Qdrant sparse + dense); chunk size sweep.

### Phase 6 — n8n workflow calling the service
- **Objective:** n8n workflows that ingest and ask through your service.
- **Concepts:** n8n nodes, expressions, credentials, the AI Agent node with a tool.
- **Docs:** [HTTP Request node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/) ·
  [Webhook node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/) ·
  [AI Agent node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/) ·
  [Ollama Chat Model node](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.lmchatollama/)
- **Commands:** open `http://localhost:5678`; export the workflow to `workflows/n8n/*.json` for git.
- **Guideline:** (a) Webhook → HTTP Request `POST http://host.docker.internal:3000/ask` → Respond to
  Webhook. (b) AI Agent with the Ollama chat model and an HTTP Request tool pointing at `/search`.
- **Done check:** `curl -X POST localhost:5678/webhook/ask -H 'content-type: application/json' -d '{"question":"..."}'` returns your
  service's answer; the agent run shows a tool call to `/search` in the execution log.
- **Stretch:** a scheduled ingest workflow that watches a folder.

### Phase 7 — Dify workflow + comparison with Dify's built-in RAG
- **Objective:** call the service from Dify three ways, and compare against Dify's own RAG.
- **Concepts:** Dify custom tools from OpenAPI, workflow Tool nodes, built-in knowledge base,
  external knowledge API.
- **Docs:** [Dify tools (Swagger/OpenAPI import)](https://docs.dify.ai/en/self-host/use-dify/workspace/tools) ·
  [Tool node](https://docs.dify.ai/en/self-host/use-dify/nodes/tools) ·
  [External knowledge base](https://docs.dify.ai/en/self-host/use-dify/knowledge/connect-external-knowledge-base) ·
  [External knowledge API spec](https://docs.dify.ai/en/self-host/use-dify/knowledge/external-knowledge-api)
- **Commands:** open `http://localhost`; import `http://host.docker.internal:3000/openapi.json`.
- **Guideline:** (a) a workflow with a Tool node calling `/ask`. (b) a `/retrieval` endpoint in Rust
  that matches Dify's external-knowledge format (Dify appends `/retrieval` and sends the API key as a
  Bearer token). (c) the same corpus in a Dify built-in knowledge base with the same Ollama models. Run
  the eval questions through all three.
- **Done check:** each of (a), (b), (c) answers in a Dify workflow run; `docs/results.md` has a
  comparison table (hit@5, judge score, latency) and the **training decision** with its reason.
- **Stretch:** Dify Agent app with your tool; per-request auth through custom headers.

### Phase 8 — Optional: LoRA fine-tune with MLX, and when it beats RAG
- **Run only if** the training decision point says so.
- **Objective:** train a LoRA adapter for a small model and measure it with the same eval.
- **Concepts:** LoRA vs full fine-tune, train/valid/test splits, overfitting, fine-tune + RAG together.
- **Docs:** [mlx-lm LoRA guide](https://github.com/ml-explore/mlx-lm/blob/main/mlx_lm/LORA.md) ·
  [Ollama import a model](https://docs.ollama.com/import)
- **Commands:** `pip install mlx-lm`;
  `mlx_lm.lora --model <hf-model> --train --data data/ft --iters 600`; `mlx_lm.fuse --model <hf-model>`
- **Guideline:** data is `train.jsonl` / `valid.jsonl` / `test.jsonl`, built from the failure cases. Keep
  test questions out of training. Python is fine here: this phase is about the model, not Rust.
- **Done check:** the eval shows the fine-tuned model + RAG vs the base model + RAG on the held-out
  set; `docs/results.md` says whether it was worth it.
- **Stretch:** serve the fused model through Ollama and select it in `/ask`.

---

## Decisions log

| Decision | Choice | Alternatives | Why |
|---|---|---|---|
| Vector DB | Qdrant | pgvector, LanceDB | user default; official Rust client; easy Docker image |
| Web framework | axum | actix-web, poem | user default; from the tokio team, so it fits the tokio learning path |
| n8n integration | HTTP Request node (+ as an AI Agent tool) | custom n8n node | no TypeScript; teaches n8n's own building blocks |
| Dify integration | OpenAPI custom tool + external knowledge API | Dify plugin (Python) | no Python plugin; the external KB gives a fair comparison with built-in RAG |
| Ollama placement | native on macOS | in Docker | Docker on Mac has no Metal GPU access |
| Dify placement | its own compose in `infra/dify/` (gitignored) | merged into our compose | Dify ships a large, versioned compose; merging it means maintaining it |
| Corpus | The Rust Book Markdown (Claude's call) | user docs | open license, on topic, easy to sanity-check; open question above |
| Fine-tuning | MLX LoRA, gated by the eval | always do it | user asked for "only if needed" |

## Research summary

- Ollama `POST /api/embed` takes `model` and `input` (string or array) and returns `embeddings` as
  `number[][]` ([docs](https://docs.ollama.com/api/embed)).
- The Qdrant Rust client connects over gRPC on port 6334 and uses builders such as
  `CreateCollectionBuilder` and `UpsertPointsBuilder` ([rust-client](https://github.com/qdrant/rust-client)).
- n8n containers reach host services through `host.docker.internal`; on Linux this needs
  `extra_hosts: host-gateway` ([docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/common-issues)).
- Dify imports OpenAPI/Swagger specs as custom tools with custom-header auth, and supports an external
  knowledge API where it appends `/retrieval` and sends a Bearer key ([tools](https://docs.dify.ai/en/self-host/use-dify/workspace/tools),
  [external KB](https://docs.dify.ai/en/self-host/use-dify/knowledge/connect-external-knowledge-base)).
- Dify self-hosting needs Docker Compose ≥ 2.24 and runs from `dify/docker` in its repo.
- mlx-lm trains with `mlx_lm.lora --train --data <dir>` (expects `train.jsonl`) and fuses with
  `mlx_lm.fuse` ([LORA.md](https://github.com/ml-explore/mlx-lm/blob/main/mlx_lm/LORA.md)).
- Local machine: arm64 Mac; Docker and Ollama are installed; Rust is **not** installed yet (phase 0).
