---
name: ml-inference-ingest
description: >
  Build OpenSearch ingest pipelines that call ML models on every document
  flowing into the index. Use this skill when the user mentions ml_inference
  ingest processor, text_embedding processor, auto-embedding documents,
  auto-classify ingest, auto-tag at index time, sentiment ingest, language
  detection ingest, NER ingest, PII redaction ingest, document enrichment,
  ingest pipeline with LLM, Bedrock connector, Bedrock Titan, Bedrock Nova
  Micro, Bedrock Claude connector, SageMaker connector, AOSS 403 Forbidden
  on _predict, AOSS error masking, or any related index-time enrichment
  topic. Activate even if the user just says "tag every doc with X" or
  "enrich documents on the way in".
compatibility: Requires Docker and uv. Bedrock connectors require AWS credentials with bedrock:InvokeModel permission. Other connectors (SageMaker, OpenAI, self-hosted) require the corresponding credentials.
metadata:
  author: opensearch-project
  version: "1.0"
---

# ML Inference Ingest

You are an OpenSearch ingest-pipeline expert. You guide users from "I want every document tagged/embedded/redacted automatically at index time" to a running ingest pipeline that does it.

## Prerequisites

- Docker installed and running (for local clusters)
- `uv` installed (for running Python scripts)
- Model credentials for the chosen connector (Bedrock IAM role, SageMaker endpoint + role, OpenAI API key, etc.)

## Optional MCP Servers

```json
{
  "mcpServers": {
    "ddg-search": {
      "command": "uvx",
      "args": ["duckduckgo-mcp-server"]
    },
    "opensearch-mcp-server": {
      "command": "uvx",
      "args": ["opensearch-mcp-server-py@latest"],
      "env": { "FASTMCP_LOG_LEVEL": "ERROR" }
    }
  }
}
```

- **`ddg-search`** — Search OpenSearch documentation. Use `search(query="site:opensearch.org <your query>")`.
- **`opensearch-mcp-server`** — Direct OpenSearch API access. Handles SigV4 auth for AOS/AOSS transparently.

### opensearch-mcp-server Configuration Variants

For basic auth (local/self-managed):
```json
{
  "opensearch-mcp-server": {
    "command": "uvx",
    "args": ["opensearch-mcp-server-py@latest"],
    "env": {
      "OPENSEARCH_URL": "<endpoint_url>",
      "OPENSEARCH_USERNAME": "<username>",
      "OPENSEARCH_PASSWORD": "<password>",
      "OPENSEARCH_SSL_VERIFY": "false",
      "FASTMCP_LOG_LEVEL": "ERROR"
    }
  }
}
```

For Amazon OpenSearch Service (AOS):
```json
{
  "opensearch-mcp-server": {
    "command": "uvx",
    "args": ["opensearch-mcp-server-py@latest"],
    "env": {
      "OPENSEARCH_URL": "<endpoint_url>",
      "AWS_REGION": "<region>",
      "AWS_PROFILE": "<profile>",
      "FASTMCP_LOG_LEVEL": "ERROR"
    }
  }
}
```

For Amazon OpenSearch Serverless (AOSS):
```json
{
  "opensearch-mcp-server": {
    "command": "uvx",
    "args": ["opensearch-mcp-server-py@latest"],
    "env": {
      "OPENSEARCH_URL": "<endpoint_url>",
      "AWS_REGION": "<region>",
      "AWS_PROFILE": "<profile>",
      "AWS_OPENSEARCH_SERVERLESS": "true",
      "FASTMCP_LOG_LEVEL": "ERROR"
    }
  }
}
```

If the cluster type is unclear, ask: "Is this a local OpenSearch cluster, Amazon OpenSearch Service, or Amazon OpenSearch Serverless?"

## Scripts

This skill reuses the shared scripts at the parent skill root:

```bash
bash scripts/start_opensearch.sh
uv run python scripts/opensearch_ops.py <command> [options]
```

See [cli-reference.md](../../cli-reference.md) for the full command reference. Key commands relevant here:
- `deploy-bedrock` / `deploy-model` — register and deploy a remote or local model
- `create-pipeline --type ingest` — create an ingest pipeline
- `create-index` — create an index, optionally with `default_pipeline` set
- `index-bulk` — bulk-load documents through the pipeline
- `search` — verify enrichment landed in the right fields

## Key Rules

- **Never deploy enrichment into production without `_simulate` first.** A misconfigured pipeline silently writes garbage into every document. See Phase 4 below.
- **Verify the model with direct `_predict` before adding the pipeline.** Eliminates 80% of debugging time when something fails.
- **Field names in `output_map` are written into `_source`.** Use simple identifiers (`language`, `sentiment`, `entities`) — not dotted paths like `ext.foo` (which become a literal dotted-key field in `_source`, not what you want).
- **`function_name` is `remote` for any connector-based model.** Hardcoded local sparse/dense models use `text_embedding`, `sparse_encoding`, etc.
- **`temperature: 0` for any classification or extraction.** Determinism matters at ingest time — same doc, same enrichment.
- **`ml_inference` ingest processor is supported on AOSS NextGen only.** Classic AOSS collections do not support it. For AOSS specifics, read [ml_inference_ingest_guide.md](ml_inference_ingest_guide.md) Section 5.

## Workflow Phases

### Phase 1 — Identify the enrichment goal

Ask the user **one** clarifying question: what enrichment do they want on every document?

Common answers:
- **Auto-embedding** for semantic/kNN search → use `text_embedding` processor (specialized) or `ml_inference` with an embedding model
- **Auto-tagging** with language, sentiment, intent, topic → `ml_inference` with a small text classifier (Nova Micro is purpose-built for this)
- **NER / entity extraction** → `ml_inference` with an LLM, prompt asks for JSON array output
- **PII redaction** → `ml_inference` with an LLM, prompt asks for redacted version
- **Summarization** of long docs → `ml_inference` with an LLM, prompt asks for a short summary

If the goal is multi-step (e.g., "language + sentiment + entities"), build a chain — each enrichment is its own `ml_inference` processor in the same pipeline. Read [ml_inference_ingest_guide.md](ml_inference_ingest_guide.md) Section 4 for the chained-pipeline pattern.

### Phase 2 — Pick a model and connector

Read [connector_patterns.md](connector_patterns.md) to choose the connector shape (Bedrock Titan, Bedrock Nova, Bedrock Claude, SageMaker, etc.) and copy the right `request_body` template. The connector is where most user errors happen — don't write one from scratch when a known-working recipe exists.

### Phase 3 — Sanity-test the model directly

Before building any pipeline:

```http
POST /_plugins/_ml/connectors/_create
{ ... connector body ... }

POST /_plugins/_ml/models/_register
{ "name": "<name>", "function_name": "remote", "connector_id": "<from above>" }

POST /_plugins/_ml/models/<model_id>/_deploy

POST /_plugins/_ml/models/<model_id>/_predict
{ "parameters": { "<key>": "<test value>" } }
```

If `_predict` fails, fix the connector before touching the pipeline. Common direct-predict failures:
- `Unrecognized parameter` → `request_body` template names a `${parameters.X}` that doesn't exist in the parameters map
- `Invalid payload` → `request_body` produces malformed JSON (often unescaped newlines, missing quotes)
- `403 Forbidden` (on AOSS) → see [ml_inference_ingest_guide.md](ml_inference_ingest_guide.md) Section 5 — AOSS masks all 4xx as 403

### Phase 4 — Build the pipeline and `_simulate`

```http
PUT /_ingest/pipeline/<name>
{
  "description": "<what this enriches>",
  "processors": [{
    "ml_inference": {
      "model_id": "<model_id>",
      "input_map":  [{ "<connector param>": "<source field name>" }],
      "output_map": [{ "<target field>": "<jsonpath into model output>" }]
    }
  }]
}

POST /_ingest/pipeline/<name>/_simulate
{
  "docs": [
    { "_index": "test", "_source": { "<source field>": "<sample value>" } }
  ]
}
```

Inspect the simulated `_source` — does the enrichment field land in the right place with the right shape? If not, fix `output_map` JSONPath before any real ingest.

### Phase 5 — Attach to an index and bulk-load

```http
PUT /<index>
{
  "settings": { "default_pipeline": "<pipeline name>" },
  "mappings": { ... fields including the enrichment fields ... }
}
```

For embedding pipelines, the target field needs `knn_vector` mapping with the right dimension.
For classification pipelines, use `keyword`.
For summarization, use `text`.

Then bulk-load and verify with a sample search:

```bash
uv run python scripts/opensearch_ops.py index-bulk --index <name> --source-file <file>
uv run python scripts/opensearch_ops.py search --index <name> --body '{"size": 1}'
```

### Phase 6 — Hand off

After the pipeline is running:

> "Your ingest pipeline is live. Each new document will be auto-enriched. Here's what you can do next:"
> 1. **If you came here from `opensearch-launchpad`**, return to its Phase 4 (Execute) — your enriched fields are ready to be queried with the search strategy you chose there.
> 2. **Build a new search app** that uses these enriched fields — use [search/opensearch-launchpad](../../search/opensearch-launchpad/SKILL.md), starting from Phase 1.
> 3. **Add query-time LLM enrichment** (query rewriting, RAG synthesis) — read [ml_inference_search_pipeline_guide.md](../../search/opensearch-launchpad/ml_inference_search_pipeline_guide.md)
> 4. **Deploy to AWS** — use [cloud/aws-setup](../../cloud/aws-setup/SKILL.md)

## Related Knowledge

- [ml_inference_ingest_guide.md](ml_inference_ingest_guide.md) — deep dive on the `ml_inference` ingest processor (substitution namespaces, output_map paths, AOSS specifics, common gotchas)
- [connector_patterns.md](connector_patterns.md) — copy-pasteable connector recipes for Bedrock Titan / Nova / Claude / Llama, SageMaker, and OpenAI
