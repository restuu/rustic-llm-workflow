# Phase 00 — Environment setup

**Depends on:** nothing · **Next:** [01-rust-basics-cli](01-rust-basics-cli.md) · **PRD:** [phase 0](../prd.md#phase-0--environment-setup)

## Context brief (cold start)

Empty repo with `CLAUDE.md`, `docs/prd.md` and these phase files. Nothing is installed for Rust yet. Docker
and Ollama are installed on this arm64 Mac. In this phase Claude may write `infra/docker-compose.yml` and
`.env.example` (non-Rust glue), after checking image tags.

## Objective

Every tool the capstone needs is running and answers a health check.

## Concepts

- rustup toolchains and components — [rustup book](https://rust-lang.github.io/rustup/)
- Ollama runs **natively** (Docker on a Mac has no Apple GPU access) — [Ollama quickstart](https://docs.ollama.com/quickstart)
- How containers reach the host: `host.docker.internal` — [n8n: reach host services](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/common-issues)
- Qdrant ports: **6333** REST + dashboard, **6334** gRPC (the Rust client) — [Qdrant quickstart](https://qdrant.tech/documentation/quickstart/)
- Dify ships its own compose file — [Dify Docker Compose](https://docs.dify.ai/en/self-host/deploy/quick-start/docker-compose)

## Steps

### 0. Your answers branch
`master` is the course; your answers never go there (see CLAUDE.md "Branches"). If you forked this repo,
clone your fork first. Docs: [git switch](https://git-scm.com/docs/git-switch)
```bash
git switch -c solutions
git branch --show-current
```
**Done check:** prints `solutions`. Commit everything from phase 00 step 1 onward on this branch.

### 1. Install Rust
Docs: [Install Rust](https://www.rust-lang.org/tools/install)
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup component add clippy rustfmt
```
**Done check:** `cargo --version && cargo clippy --version` both print versions.

### 2. Pick and pull Ollama models
Docs: [Ollama library](https://ollama.com/library) · [/api/embed](https://docs.ollama.com/api/embed)

Pick one embedding model and one small chat model that fit your Mac's memory. The chat model must
support **tool calling** (the Ollama library marks these with a "tools" tag): the n8n AI Agent needs it
in phase 06.
```bash
ollama pull <embedding-model>
ollama pull <chat-model>
curl -s localhost:11434/api/embed -d '{"model":"<embedding-model>","input":"hi"}' | jq '.embeddings[0] | length'
```
**Done check:** the last command prints a number (the dimension). Note it: Qdrant needs it in phase 4.

### 3. Start Qdrant and n8n
Ask Claude to write `infra/docker-compose.yml` with `qdrant` (6333, 6334, named volume) and `n8n` (5678,
named volume, `extra_hosts: ["host.docker.internal:host-gateway"]`), and a `.env.example` that lists
**every** variable later phases use:

| Variable | Used in | Example |
|---|---|---|
| `OLLAMA_URL` | 02 | `http://localhost:11434` |
| `EMBED_MODEL`, `CHAT_MODEL`, `EMBED_DIM` | 02, 04 | from step 2 |
| `QDRANT_URL`, `QDRANT_COLLECTION` | 04 | `http://localhost:6334`, `rust_book` |
| `ANTHROPIC_API_KEY`, `CLAUDE_MODEL` | 05 | your key (only in `.env`) |
| `RAG_API_KEY` | 07 | any random string |

Then `cp .env.example .env` and fill it in.
```bash
docker compose -f infra/docker-compose.yml up -d
docker compose -f infra/docker-compose.yml ps
```
**Done check:** `curl -s localhost:6333/collections` returns JSON with `"status":"ok"`, and
`http://localhost:5678` shows the n8n owner setup page.

### 4. Start Dify
Docs: [Dify Docker Compose](https://docs.dify.ai/en/self-host/deploy/quick-start/docker-compose). Needs
Docker Compose ≥ 2.24 (`docker compose version`).

You already have a fork at `~/github.com/restuu/dify`. Use it only if it's close to the latest upstream
release; otherwise clone the latest release into `infra/dify/` (gitignored):
```bash
git clone --branch "$(curl -s https://api.github.com/repos/langgenius/dify/releases/latest | jq -r .tag_name)" https://github.com/langgenius/dify.git infra/dify
cd infra/dify/docker && cp .env.example .env && docker compose up -d
```
**Done check:** `http://localhost/install` shows the Dify admin setup page.

### 5. Reachability from containers
Docs: [Ollama FAQ](https://docs.ollama.com/faq)
```bash
docker compose -f infra/docker-compose.yml exec n8n node -e "fetch('http://host.docker.internal:11434/api/version').then(r=>r.text()).then(console.log)"
```
(The n8n image may not include `wget` or `curl`; `node` is always there.)
**Done check:** prints a JSON version. If not, set `OLLAMA_HOST=0.0.0.0` as in the FAQ and restart Ollama.

## Phase done check

All five step checks pass. Optional: put them in `scripts/check-env.sh` so later sessions can run one command.

## Stretch

- `scripts/check-env.sh` that exits non-zero on the first failing check.
- Read the Dify `docker-compose.yaml` and list which vector DB Dify uses by default (you'll compare in phase 7).

## Don't

- Don't run Ollama in Docker on the Mac.
- Don't commit `.env` or `infra/dify/`.
