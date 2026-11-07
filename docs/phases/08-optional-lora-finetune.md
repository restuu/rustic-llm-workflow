# Phase 08 — Optional: LoRA fine-tune with MLX, and when it beats RAG

**Depends on:** [07](07-dify-workflow.md) **and** "Training decision: yes" in `docs/results.md` · **PRD:** [phase 8](../prd.md#phase-8--optional-lora-fine-tune-with-mlx-and-when-it-beats-rag)

## Context brief (cold start)

Skip this phase if `docs/results.md` says "Training decision: no". Otherwise, the eval showed answers
that retrieval can't fix (format/style, or domain vocabulary). This phase trains a LoRA adapter for a
small model on Apple silicon with mlx-lm, and measures it with the same eval. Python is fine here: this
phase is about the model, not Rust.

## Objective

A held-out eval shows fine-tuned model + RAG vs base model + RAG, and `docs/results.md` says whether it
was worth it.

## Concepts

- LoRA / QLoRA, adapters, fuse — [mlx-lm LoRA guide](https://github.com/ml-explore/mlx-lm/blob/main/mlx_lm/LORA.md)
- Data format: `train.jsonl`, `valid.jsonl`, `test.jsonl` — same guide, "Data" section
- Overfitting and held-out data — keep every eval question out of training
- Serving the result in Ollama — [Ollama import](https://docs.ollama.com/import)

## Steps

### 1. Environment
```bash
python3 -m venv .venv && source .venv/bin/activate
pip install mlx-lm
mlx_lm.lora --help
```
**Done check:** the help text prints. (`.venv/` should be added to `.gitignore`.)

### 2. Build the data
From the failure cases in `docs/results.md`, write 100+ training examples in `data/ft/train.jsonl`, a
small `valid.jsonl`, and a `test.jsonl` built only from held-out eval questions. Include retrieved context
in the prompt if you'll use the model with RAG.
**Done check:** no eval question id appears in `train.jsonl` or `valid.jsonl`.

### 3. Train and fuse
Pick a small base model available for MLX that matches your Ollama chat model family. If it is a
quantized (e.g. 4-bit) model, training uses QLoRA, and the fuse step needs `--dequantize` so Ollama can
import the result.
```bash
mlx_lm.lora --model <hf-model> --train --data data/ft --iters 600
mlx_lm.lora --model <hf-model> --adapter-path adapters --data data/ft --test
mlx_lm.fuse --model <hf-model> --dequantize   # drop --dequantize for an unquantized base
```
**Done check:** validation loss goes down; the `--test` run prints test loss/perplexity.

### 4. Measure
Import the fused model into Ollama ([Ollama import](https://docs.ollama.com/import)) under a new name,
set `CHAT_MODEL` to it, and use `/ask` with `"model":"ollama"` — no Rust change needed. Rerun the eval on
the held-out questions only, once with the base model and once with the fine-tuned one.
(`mlx_lm.server` would also work but needs a new `ChatModel` implementation; skip it.)
**Done check:** `docs/results.md` has base + RAG vs fine-tuned + RAG on the same held-out questions, and
a one-paragraph verdict.

## Stretch

- DoRA (`--fine-tune-type dora`) vs LoRA on the same data.
- Fine-tuned model without RAG: does it still need retrieval? (Expected: yes, for facts.)
