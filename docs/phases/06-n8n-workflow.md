# Phase 06 — n8n workflow calling the service

**Depends on:** [05](05-rag-ask-eval.md) · **Can run before or after:** [07](07-dify-workflow.md) · **PRD:** [phase 6](../prd.md#phase-6--n8n-workflow-calling-the-service)

## Context brief (cold start)

`rag-service` runs on the host at port 3000 with `/search` and `/ask`. n8n runs in Docker at
`localhost:5678` and reaches the host at `http://host.docker.internal:3000`. This phase is no-code: you
build workflows in the n8n editor and export them to git. Claude may write workflow JSON if you ask.

## Objective

An n8n webhook answers questions through your service, and an n8n AI Agent uses `/search` as a tool.

## Concepts

- Nodes, connections, expressions — [n8n workflows](https://docs.n8n.io/workflows/) · [expressions](https://docs.n8n.io/code/expressions/)
- [Webhook node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/) (test URL vs production URL)
- [HTTP Request node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/)
- [Respond to Webhook node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.respondtowebhook/)
- [AI Agent node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/) · [Ollama Chat Model](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.lmchatollama/) · [HTTP Request node as an agent tool](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolhttprequest/)

## Steps

### 1. Reach the service from n8n
```bash
cargo run -p rag-service
docker compose -f infra/docker-compose.yml exec n8n node -e "fetch('http://host.docker.internal:3000/health').then(r=>r.text()).then(console.log)"
```
**Done check:** prints the health response.

### 2. Workflow A: webhook → `/ask`
Webhook (POST, path `ask`, respond "Using Respond to Webhook node") → HTTP Request (POST
`http://host.docker.internal:3000/ask`, JSON body from `{{ $json.body.question }}`) → Respond to Webhook.
Activate the workflow, then:
```bash
curl -s -X POST localhost:5678/webhook/ask -H 'content-type: application/json' -d '{"question":"What is a slice?"}' | jq
```
**Done check:** the response contains your service's `answer` and `sources`. (`/webhook-test/ask` works
only while the editor is listening; `/webhook/ask` needs the workflow active.)

### 3. Workflow B: AI Agent with your search as a tool
Chat Trigger → AI Agent, with the Ollama Chat Model (base URL `http://host.docker.internal:11434`, your
tool-calling chat model from phase 00) and an HTTP Request node attached as a tool that calls `/search`. Write a clear tool description: the agent decides when to call it
from that text.

**Done check:** ask "how do I return an error from main?" in the chat panel; the execution log shows a
tool call to `/search` and the answer cites a chapter.

### 4. Save to git
Export both workflows (⋯ → Download) into `workflows/n8n/`.
**Done check:** `ls workflows/n8n/*.json` lists two files, and no secret values were exported: n8n
exports only credential references, so
`jq -c '[.nodes[].credentials // {} | .[] | keys] | flatten | unique' workflows/n8n/*.json`
prints `["id","name"]` (or `[]`) for each file.

## Phase done check

Steps 2 and 3 pass, and the exported workflows are in git.

## Stretch

- A scheduled workflow that calls `/ingest` when a folder changes.
- Run 10 eval questions through workflow A with a Loop node and compare latency with direct `/ask`.
