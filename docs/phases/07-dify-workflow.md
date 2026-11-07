# Phase 07 — Dify workflow + comparison with Dify's built-in RAG

**Depends on:** [05](05-rag-ask-eval.md) · **Can run before or after:** [06](06-n8n-workflow.md) · **Next:** [08](08-optional-lora-finetune.md) only if the decision below says so · **PRD:** [phase 7](../prd.md#phase-7--dify-workflow--comparison-with-difys-built-in-rag)

## Context brief (cold start)

`rag-service` has `/search`, `/ask` and `/openapi.json`; the eval set and baseline are in
`eval/questions.jsonl` and `docs/results.md`. Dify runs from its own compose at `http://localhost`. This
phase connects Dify three ways, compares them on the same questions, and makes the training decision.
One small Rust addition: a `/retrieval` endpoint in Dify's format, with Bearer-key auth. Two sessions are
likely: (A) tool + `/retrieval`, (B) built-in KB + comparison.

## Objective

Three Dify setups answer the eval questions — (a) custom tool → `/ask`, (b) external knowledge base →
`/retrieval`, (c) Dify's built-in knowledge base — and `docs/results.md` compares them and records the
training decision.

## Concepts

- Custom tools from OpenAPI — [Dify tools](https://docs.dify.ai/en/self-host/use-dify/workspace/tools) · [Tool node](https://docs.dify.ai/en/self-host/use-dify/nodes/tools)
- External knowledge — [connect](https://docs.dify.ai/en/self-host/use-dify/knowledge/connect-external-knowledge-base) · [API spec](https://docs.dify.ai/en/self-host/use-dify/knowledge/external-knowledge-api)
- Built-in knowledge base — [Dify knowledge](https://docs.dify.ai/en/self-host/use-dify/knowledge/readme) · [Knowledge base API](https://docs.dify.ai/en/self-host/use-dify/knowledge/manage-knowledge/maintain-dataset-via-api) (has a retrieve endpoint for scoring setup c)
- Auth as an axum extractor or middleware — [axum middleware](https://docs.rs/axum/latest/axum/middleware/index.html)

## Session A

### 1. Ollama as a Dify model provider
In Dify: Settings → Model Provider → install Ollama, base URL `http://host.docker.internal:11434`, add your
chat and embedding models.
**Done check:** a blank Chatflow answers with your Ollama chat model.

### 2. (a) Custom tool → `/ask`
Tools → Custom → Create, import `http://host.docker.internal:3000/openapi.json`. Build a Workflow: Start
(input `question`) → Tool node (`ask`) → End.
**Done check:** a workflow run returns your service's answer and sources.

### 3. (b) `/retrieval` endpoint in Rust (test first)
Spec: Dify POSTs `{knowledge_id, query, retrieval_setting: {top_k, score_threshold}}` with
`Authorization: Bearer <key>` and expects `{"records":[{"content","score","title","metadata"}]}`
(`metadata` must be an object, never `null`). Any non-200 shows an error in Dify; the docs suggest body
codes 1001/1002 but no HTTP status. **Our choice:** 403 for a missing or wrong key.

Ask Claude for failing tests: missing/invalid Bearer → 403 with `error_code`; `top_k` respected; records
below `score_threshold` dropped; empty result → `{"records": []}`. Hint: Dify wants scores in 0–1 —
check what range your Qdrant distance returns.

Then in Dify: Knowledge → External Knowledge API → endpoint `http://host.docker.internal:3000` (Dify
appends `/retrieval`), API key from your `.env`. Connect an external knowledge base with any knowledge ID.
```bash
cargo test -p rag-service
set -a; source .env; set +a
curl -s localhost:3000/retrieval -H "authorization: Bearer $RAG_API_KEY" -H 'content-type: application/json' \
  -d '{"knowledge_id":"rust-book","query":"what is a slice","retrieval_setting":{"top_k":3,"score_threshold":0.0}}' | jq '.records | length'
```
**Done check:** tests pass; curl prints `3`; Dify's "Retrieval Testing" page returns chunks.

## Session B

### 4. (c) Built-in knowledge base
Upload the same `data/rust-book/src/*.md` to a Dify knowledge base, using your Ollama embedding model.
Write down Dify's chunk settings and its vector DB.
**Done check:** Dify's Retrieval Testing returns the ownership chapter for "what is a borrow".

### 5. Compare and decide
Run the eval questions through (a), (b) and (c): through the Dify API, or by hand for a sample of 15 if
that's faster (say which in the results). Record judge score and latency for all three, and hit@5:
- (a) and (b) use your own retrieval, so their hit@5 is your phase 05 number.
- (c): call the knowledge base API's retrieve endpoint for each question with `top_k: 5`, and check the
  returned document names against `expected_sources`. A small script is fine (Claude may write it; it's
  not Rust).

Then the **training decision** (PRD "Training decision point"): fine-tune only if retrieval is good
(hit@5 ≥ target) but answers fail on format/style or domain vocabulary after prompt fixes. If retrieval
is weak, the next step is better chunking/reranking, not phase 08.

**Done check:** `docs/results.md` has the comparison table and a "Training decision: yes/no, because …" line with numbers.

## Stretch

- A Dify Agent app that uses your tool and decides when to call it.
- Per-caller auth: pass a header through with `{{request.headers.X-...}}` (see the tools doc).
