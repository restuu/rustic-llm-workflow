# Phase 05 — RAG `/ask` endpoint + evaluation set

**Depends on:** [04](04-ingest-qdrant.md) · **Next:** [06-n8n-workflow](06-n8n-workflow.md) and/or [07-dify-workflow](07-dify-workflow.md) · **PRD:** [phase 5](../prd.md#phase-5--rag-ask--evaluation-set)

## Context brief (cold start)

The corpus is in Qdrant and `/search` works. This phase adds generation (`/ask`, with sources), a Claude
`ChatModel` to compare against Ollama, and an evaluation CLI. The numbers from this phase decide the
chunking settings and, in phase 07, whether fine-tuning is needed. You write the Rust.

This phase is three sessions: **A** `/ask` + Claude client, **B1** write the eval set, **B2** eval CLI.

## Objective

`/ask` returns `answer` + `sources` with either model, and `rag-cli eval` prints hit@5, MRR and a judge
score for both.

## Concepts

- Trait objects to switch models at runtime — [Book 18.2](https://doc.rust-lang.org/book/ch18-02-trait-objects.html)
- Ollama chat — [/api/chat](https://docs.ollama.com/api/chat) · Claude — [Messages API](https://docs.anthropic.com/en/api/messages) · [models](https://docs.anthropic.com/en/docs/about-claude/models/overview)
- Grounded prompts and citations — [Anthropic: reduce hallucinations](https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/reduce-hallucinations)
- Eval metrics: **hit@k** = share of questions where an expected source is in the top k; **MRR** = mean of
  1/rank of the first correct source (0 if none)
- LLM-as-judge — [Anthropic: create evals](https://docs.anthropic.com/en/docs/test-and-evaluate/develop-tests)
- Reading JSON Lines — [std::io::BufRead::lines](https://doc.rust-lang.org/std/io/trait.BufRead.html#method.lines)

## Session A — `/ask`

### 1. Claude `ChatModel`
Pick the Claude model from the models page. Put `ANTHROPIC_API_KEY` in `.env` (never in code, never in
git). Ask Claude for failing tests (fake server again): headers sent (`x-api-key`, `anthropic-version`),
body shape, parsing the text block, 429/529 → a retryable error variant.

### 2. `/ask`
Retrieve top-k → build a prompt that tells the model to answer only from the context and cite sources →
generate. Request: `{"question": "...", "model": "ollama" | "claude", "top_k": 5}`.
```bash
curl -s localhost:3000/ask -H 'content-type: application/json' -d '{"question":"What does the ? operator do?","model":"ollama"}' | jq
curl -s localhost:3000/ask -H 'content-type: application/json' -d '{"question":"What does the ? operator do?","model":"claude"}' | jq
```
**Done check:** both return `answer` and a non-empty `sources` list that includes the error-handling chapter.

### 3. Add `/ask` to the OpenAPI spec
Give it an `operationId` (`ask`) and describe the request fields: Dify (phase 07) and the n8n agent
(phase 06) read these descriptions.
```bash
curl -s localhost:3000/openapi.json | jq '.paths | keys'
```
**Done check:** the list includes `/ask`, and `.servers[0].url` is `http://host.docker.internal:3000`.

## Session B1 — write the eval set

### 4. Write the eval set
`eval/questions.jsonl`, 30–50 lines, one object per line:
`{"id":"q01","question":"...","expected_sources":["ch09-02-recoverable-errors-with-result.md"]}`.
Write them yourself. Claude may draft candidates; you check each against the book. Mix easy lookups, two-
chapter questions and a few with no answer in the corpus (expected: the model says it doesn't know).
`expected_sources` are file names only, matching `source` from `/search`.

**Done check:** `jq -c . eval/questions.jsonl | wc -l` prints 30–50, and every `expected_sources` file
exists: `jq -r '.expected_sources[]' eval/questions.jsonl | sort -u | while read f; do test -f data/rust-book/src/$f || echo "missing $f"; done` prints nothing.

## Session B2 — eval CLI

### 5. Eval CLI
Ask Claude for failing tests for the metric functions first (hit@k and MRR on tiny hand-made inputs).
```bash
cargo run -p rag-cli -- eval eval/questions.jsonl --k 5 --model ollama
cargo run -p rag-cli -- eval eval/questions.jsonl --k 5 --model claude --judge claude
```
**Done check:** prints hit@5, MRR and judge score (share of answers "supported by sources"). Write the
baseline into `docs/results.md` with the date, models and chunk settings.

## Phase done check

`cargo test --workspace` green, and `docs/results.md` has the baseline for both models. Compare with the
PRD targets (hit@5 ≥ 0.8, MRR ≥ 0.6, judge ≥ 80%). If you change a target, log why in the PRD decisions log.

## Stretch

- Reranking: score the top 20 with the chat model or a cross-encoder, keep 5; measure the change.
- Hybrid search (dense + sparse) — [Qdrant hybrid queries](https://qdrant.tech/documentation/concepts/hybrid-queries/).
- Sweep chunk size / overlap and plot hit@5.
