# rustic-llm-workflow

Learn Rust the hard way by building one real project: a RAG service (ingest → chunk → embed → store →
retrieve → generate) written in Rust and called from [n8n](https://docs.n8n.io) and
[Dify](https://docs.dify.ai) workflows. Models run locally with [Ollama](https://docs.ollama.com), with
[Claude](https://docs.anthropic.com) as a hosted comparison.

This repo is a **course, not a solution**. It holds the plan and the rules. You write the code.

## How it works

- Each phase is one file in [`docs/phases/`](docs/phases/), sized for one session. It gives doc links,
  exact commands, and a **done check** (a test, a `curl` with expected output, or a workflow run).
- Your AI assistant is a tutor, not a ghostwriter. [`CLAUDE.md`](CLAUDE.md) (also [`AGENTS.md`](AGENTS.md)
  for Codex, Cursor and others) tells it to give **guidance, never the direct answer**: concepts, doc
  links, hints and failing tests that define "done". The Rust is yours.
- Only primary docs are linked: The Rust Book, std, docs.rs, tokio, and each tool's official docs.

## Start

1. Fork this repo, then clone your fork.
2. Put your work on a branch. `master` is the course and stays clean:
   ```bash
   git switch -c solutions
   ```
3. Open [`docs/phases/00-env-setup.md`](docs/phases/00-env-setup.md) and follow it.

To pull later course fixes into your fork, merge them into `master`, then merge `master` into `solutions`:
```bash
git remote add upstream https://github.com/restuu/rustic-llm-workflow.git
git switch master && git pull upstream master
git switch solutions && git merge master
```

## Phases

| # | Phase | You end with |
|---|-------|--------------|
| 00 | [Environment setup](docs/phases/00-env-setup.md) | Rust, Ollama, Qdrant, n8n, Dify running |
| 01 | [Rust basics: Rustlings + a chunking CLI](docs/phases/01-rust-basics-cli.md) | a tested chunker |
| 02 | [Async HTTP client for Ollama](docs/phases/02-ollama-client.md) | embed + chat from Rust |
| 03 | [axum service with /embed and /search](docs/phases/03-axum-service.md) | an HTTP service + OpenAPI spec |
| 04 | [Ingest pipeline + Qdrant](docs/phases/04-ingest-qdrant.md) | The Rust Book indexed in Qdrant |
| 05 | [RAG /ask + evaluation set](docs/phases/05-rag-ask-eval.md) | answers with sources, hit@5 / MRR / judge scores |
| 06 | [n8n workflow](docs/phases/06-n8n-workflow.md) | a webhook and an AI agent using your service |
| 07 | [Dify workflow + comparison](docs/phases/07-dify-workflow.md) | your RAG vs Dify's, and a fine-tune decision |
| 08 | [Optional: LoRA fine-tune with MLX](docs/phases/08-optional-lora-finetune.md) | only if phase 07 says it's needed |

The why, the targets and the decisions behind the plan are in [`docs/prd.md`](docs/prd.md).

## Requirements

- The phases are written for an Apple silicon Mac: Ollama runs natively for the GPU, and phase 08 uses
  MLX. Other platforms are untested.
- Docker, plus disk space for the Ollama models and the Dify images.
- An Anthropic API key is optional (only for the Claude comparison in phase 05).

## Non-goals

No production deployment, no cloud, no Kubernetes, no frontend.

## License

The course material in this repo is licensed under [CC BY 4.0](LICENSE): copy, adapt and teach from it,
including commercially, as long as you credit "restuu/rustic-llm-workflow" and link back. Code you write
in your own fork is yours, under any license you choose.
