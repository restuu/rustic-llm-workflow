# Graph Report - rustic-llm-workflow  (2026-11-07)

## Corpus Check
- Corpus is ~8,990 words - fits in a single context window. You may not need a graph.

## Summary
- 113 nodes · 275 edges · 8 communities (7 shown, 1 thin omitted)
- Extraction: 97% EXTRACTED · 3% INFERRED · 0% AMBIGUOUS · INFERRED: 9 edges (avg confidence: 0.82)
- Token cost: 0 input · 0 output

## Community Hubs (Navigation)
- Rust Basics and Async Traits
- Environment Setup
- n8n and Dify Workflows
- axum Service
- RAG Ask and Evaluation
- Learning Mode Rules
- Fine-tune and Search Data
- Ingest and Qdrant

## God Nodes (most connected - your core abstractions)
1. `PRD: rustic-llm-workflow — learn Rust by building a RAG service for n8n and Dify` - 41 edges
2. `Phase 05 — RAG /ask endpoint + evaluation set` - 20 edges
3. `rustic-llm-workflow` - 18 edges
4. `Phase 03 — axum service with /embed and /search` - 18 edges
5. `rag-service` - 17 edges
6. `Phase 02 — Async HTTP client for Ollama` - 16 edges
7. `Phase 04 — Ingest pipeline + Qdrant` - 15 edges
8. `Phase 07 — Dify workflow + comparison with Dify's built-in RAG` - 15 edges
9. `Phase 00 — Environment setup` - 14 edges
10. `Phase 06 — n8n workflow calling the service` - 13 edges

## Surprising Connections (you probably didn't know these)
- `What we're NOT building` --semantically_similar_to--> `Non-goals (no production, cloud, k8s, frontend)`  [INFERRED] [semantically similar]
  docs/prd.md → CLAUDE.md
- `rustic-llm-workflow` --references--> `Phase 00 — Environment setup`  [EXTRACTED]
  CLAUDE.md → docs/phases/00-env-setup.md
- `rustic-llm-workflow` --references--> `Phase 01 — Rust basics: Rustlings + a chunking CLI`  [EXTRACTED]
  CLAUDE.md → docs/phases/01-rust-basics-cli.md
- `rustic-llm-workflow` --references--> `Phase 02 — Async HTTP client for Ollama`  [EXTRACTED]
  CLAUDE.md → docs/phases/02-ollama-client.md
- `rustic-llm-workflow` --references--> `Phase 03 — axum service with /embed and /search`  [EXTRACTED]
  CLAUDE.md → docs/phases/03-axum-service.md

## Import Cycles
- None detected.

## Hyperedges (group relationships)
- **RAG pipeline: ingest → chunk → embed → store → retrieve → generate** — docs_phases_04_ingest_qdrant_ingest_pipeline, docs_phases_01_rust_basics_cli_chunker, docs_prd_embedder_trait, docs_prd_vectorstore_trait, docs_prd_post_search, docs_prd_post_ask [EXTRACTED 1.00]
- **Evaluation harness (hit@k, MRR, judge over eval set)** — docs_phases_05_rag_ask_eval_eval_cli, docs_phases_05_rag_ask_eval_evaluation_set_eval_questions_jsonl, docs_phases_05_rag_ask_eval_hit_k, docs_phases_05_rag_ask_eval_mrr, docs_phases_05_rag_ask_eval_llm_as_judge, docs_prd_docs_results_md [EXTRACTED 1.00]
- **Dify three-way comparison feeding training decision** — docs_phases_07_dify_workflow_dify_custom_tool_from_openapi, docs_phases_07_dify_workflow_dify_external_knowledge_api, docs_phases_07_dify_workflow_dify_built_in_knowledge_base, docs_phases_07_dify_workflow_rag_comparison_a_b_c, docs_prd_training_decision_point [EXTRACTED 1.00]

## Communities (8 total, 1 thin omitted)

### Community 0 - "Rust Basics and Async Traits"
Cohesion: 0.16
Nodes (22): Chunker, Ownership and borrowing, Phase 01 — Rust basics: Rustlings + a chunking CLI, Result and ? operator, Rustlings, UTF-8 byte vs char index, async/await, Fake HTTP server tests (+14 more)

### Community 1 - "Environment Setup"
Cohesion: 0.18
Nodes (17): .env.example, Phase 00 — Environment setup, Qdrant ports 6333 REST / 6334 gRPC, rustup, host.docker.internal, infra/docker-compose.yml, PRD Phase 0 — Environment setup, PRD Phase 1 — Rust basics: Rustlings + small CLI (+9 more)

### Community 2 - "n8n and Dify Workflows"
Cohesion: 0.23
Nodes (17): Tool-calling chat model, n8n AI Agent node, n8n HTTP Request node, n8n Webhook node, Phase 06 — n8n workflow calling the service, Workflow export to workflows/n8n, Bearer API key auth, Dify built-in knowledge base (+9 more)

### Community 3 - "axum Service"
Cohesion: 0.23
Nodes (13): Custom error enum with From, Cosine similarity, IntoResponse error mapping, operationId, Phase 03 — axum service with /embed and /search, Router tests with tower oneshot, Shared state with Arc, axum (+5 more)

### Community 4 - "RAG Ask and Evaluation"
Cohesion: 0.23
Nodes (12): Baseline, Eval CLI, Grounded prompt with citations, hit@k, LLM-as-judge, MRR, Phase 05 — RAG /ask endpoint + evaluation set, Retryable error (429/529) (+4 more)

### Community 6 - "Fine-tune and Search Data"
Cohesion: 0.25
Nodes (9): Search response shape, Qdrant payload (source, heading, text), Evaluation set (eval/questions.jsonl), Held-out test split, LoRA / QLoRA, mlx_lm.fuse, Ollama model import, Phase 08 — Optional: LoRA fine-tune with MLX, and when it beats RAG (+1 more)

### Community 7 - "Ingest and Qdrant"
Cohesion: 0.32
Nodes (7): Bounded concurrency, Heading-aware vs fixed-size chunking, Ingest pipeline, Phase 04 — Ingest pipeline + Qdrant, POST /ingest, The Rust Book corpus, VectorStore trait

## Knowledge Gaps
- **16 isolated node(s):** `rustup`, `.env.example`, `Rustlings`, `Ownership and borrowing`, `Result and ? operator` (+11 more)
  These have ≤1 connection - possible missing edges or undocumented components. (Counts symbols only; 21 node(s) total have ≤1 connection when file, concept and rationale nodes are included.)
- **1 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `PRD: rustic-llm-workflow — learn Rust by building a RAG service for n8n and Dify` connect `Environment Setup` to `Rust Basics and Async Traits`, `n8n and Dify Workflows`, `axum Service`, `RAG Ask and Evaluation`, `Learning Mode Rules`, `Fine-tune and Search Data`, `Ingest and Qdrant`?**
  _High betweenness centrality (0.313) - this node is a cross-community bridge._
- **Are the 2 inferred relationships involving `rag-service` (e.g. with `rag-clients` and `rag-core`) actually correct?**
  _`rag-service` has 2 INFERRED edges - model-reasoned connections that need verification._
- **What connects `rustup`, `.env.example`, `Rustlings` to the rest of the system?**
  _16 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Why does `rustic-llm-workflow` connect `Learning Mode Rules` to `Rust Basics and Async Traits`, `Environment Setup`, `n8n and Dify Workflows`, `axum Service`, `RAG Ask and Evaluation`, `Fine-tune and Search Data`, `Ingest and Qdrant`?**
  _High betweenness centrality (0.229) - this node is a cross-community bridge._
- **Why does `Phase 05 — RAG /ask endpoint + evaluation set` connect `RAG Ask and Evaluation` to `Rust Basics and Async Traits`, `Environment Setup`, `n8n and Dify Workflows`, `Learning Mode Rules`, `Fine-tune and Search Data`, `Ingest and Qdrant`?**
  _High betweenness centrality (0.144) - this node is a cross-community bridge._