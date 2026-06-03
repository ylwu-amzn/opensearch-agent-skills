# ML Inference Search Pipeline Guide

Use this guide when the user wants to call a machine learning model **at query time or response time** — query rewriting, intent detection, multilingual routing, query expansion, LLM re-ranking, or grounded RAG answer synthesis. The same `ml_inference` processor handles all of these via two variants: **search request** processor and **search response** processor.

> Related guides
> - For ingest-time enrichment (auto-embeddings, classification at index time), see [`ingestion/ml-inference-ingest/`](../../ingestion/ml-inference-ingest/SKILL.md) instead.
> - For the simpler pre-built `neural` query (text → kNN), see [`opensearch_semantic_search_guide.md`](opensearch_semantic_search_guide.md). Use `ml_inference` request processor only when you need behavior `neural` query can't express.
> - For agentic search (LLM-driven multi-step retrieval), see [`agentic_search_guide.md`](agentic_search_guide.md). `ml_inference` is a lighter, single-shot tool — use it when you need predictable inline LLM enrichment, not multi-step planning.

---

## 1. The three substitution namespaces

The `ml_inference` processor has three places where templates substitute values, each with its own scope. Mixing them up is the most common configuration error.

| Namespace | Where it appears | Source of values |
|---|---|---|
| `${parameters.X}` | Inside the **connector's** `request_body` template | Connector's static `parameters` map + per-request parameters (from `input_map` keys) |
| `${input_map.X}` | Inside the processor's `model_input` field (open-source) — **not supported on AOSS NextGen** | The JSONPath value extracted by `input_map[X]` |
| `${X}` (where `X` is an `output_map` key) | Inside the processor's `query_template` (request processor) or `model_config.prompt` (response processor) | The model's prediction output, looked up by the JSONPath in `output_map[X]` |

**These do not overlap.** `output_map` keys are never visible in the connector body. `input_map` keys are never visible in `query_template`. Most user errors come from forgetting this.

---

## 2. Search request processor — query rewriting

### When to use

The user wants the pipeline to **modify the query** based on an LLM call before retrieval runs. Common patterns:

- Detect query language → add a `language` filter (multilingual routing)
- Classify intent → choose which index/filter/boost set to apply
- Spell-correct or expand query → rewrite the `must` clause
- Extract entities from the query → add structured filters

### Two ways to wire it

**Pattern A — `query_template` rewrite** (the model output drives the rewritten query)

The pipeline replaces the user's query body entirely with `query_template`, substituting model output via `${output_map_key}`. Best when the model produces a vector or score the rewritten query directly consumes (e.g. NL → kNN with a Titan embedding).

```http
PUT /_search/pipeline/nl-to-knn
{
  "request_processors": [{
    "ml_inference": {
      "model_id": "<titan_embed_model>",
      "function_name": "remote",
      "input_map":  [{ "inputText": "query.match.title.query" }],
      "output_map": [{ "query_vector": "$.embedding" }],
      "query_template": "{\"size\":3,\"query\":{\"knn\":{\"title_embedding\":{\"vector\":${query_vector},\"k\":3}}}}"
    }
  }]
}
```

User sends `{ "query": { "match": { "title": { "query": "space exploration" } } } }`. Pipeline embeds it and replaces the whole query with a kNN over `title_embedding`.

**Pattern B — `output_map` JsonPath direct rewrite** (overwrite an existing slot)

The pipeline writes the model output to a specific path inside the existing query body. The user's query stays mostly intact — only the targeted slot is overwritten. Best for adding filters or modifying boost values where the user controls the outer query shape.

```http
PUT /_search/pipeline/lang-route
{
  "request_processors": [{
    "ml_inference": {
      "model_id": "<lang_detect_model>",
      "function_name": "remote",
      "input_map":  [{ "text": "query.bool.must[0].match.text.query" }],
      "output_map": [{ "query.bool.filter[0].term.language.value": "output.message.content[0].text" }]
    }
  }]
}
```

The user must provide a placeholder filter slot in their request:
```json
{
  "query": {
    "bool": {
      "must":   [{ "match": { "text": { "query": "Carlos en Madrid" } } }],
      "filter": [{ "term":  { "language": "PLACEHOLDER" } }]
    }
  }
}
```

The pipeline overwrites `"PLACEHOLDER"` with the detected language code. The literal string `"PLACEHOLDER"` doesn't matter — anything works as long as the JSONPath resolves to an existing value the processor can `JsonPath.set(...)` into. This is enforced by `validateRequiredOutputMappingFields` — paths starting with `query.*` must already exist.

### Critical gotchas

1. **`input_map` key must match the connector's `${parameters.X}` placeholder.** If the connector's `request_body` template has `${parameters.text}`, the pipeline must populate a parameter named `text`. A mismatch surfaces (on open-source) as `Invalid payload: ... parameter placeholder not filled in payload: text`. On AOSS NextGen, the same error gets remapped to a generic `403 Forbidden` (see `MachineLearningRestClient.handleResponseException` 4xx → 403 remap).

2. **Don't extract from a parent match clause.** `query.bool.must[0].match.text` returns the entire normalized match clause object (with `auto_generate_synonyms_phrase_query`, `boost`, etc.) once OpenSearch has parsed the request. JSONPath needs to land on the leaf string: `query.bool.must[0].match.text.query` (note the trailing `.query`).

3. **`query_template` only sees `output_map` keys.** `${input_map.X}` substitution is supported only inside `model_input` — and `model_input` is **not supported on AOSS NextGen**. If you need to replay the user's text in the rewritten query alongside the model output, use Pattern B (write into placeholder slots) instead of Pattern A.

4. **AOSS NextGen does not register the `ext.ml_inference` extension.** Use `$._request.<jsonpath>` syntax (see Section 3) inside response processors instead — `ext.ml_inference` works only on managed/self-hosted OpenSearch.

---

## 3. Search response processor — RAG and per-hit enrichment

### When to use

The user wants the pipeline to **call an LLM after retrieval** with the hits as context, and attach the LLM's output to the search response. Common patterns:

- Cross-document RAG synthesis (one answer for the whole result set)
- LLM re-ranking (per-hit relevance score)
- Per-hit summarization or classification
- Citation extraction

### `one_to_one` controls the fundamental shape

| Setting | Behavior | Use it for |
|---|---|---|
| `one_to_one: false` (default) | One model call sees ALL hits. The processor aggregates each hit's `input_map` field into a `List<String>`. Use `${parameters.X.toString()}` to embed the list as a single string slot, OR use `${parameters.X}` (no quotes around it in the connector body) when the model's API expects a JSON array directly. Output is written either as a single value replicated across hits (string output) or fanned out per hit using `results[*]` indexing (array-aligned output). | **Cross-document synthesis** (RAG, summarization across results) AND **batch per-hit scoring** (rerank, where the model returns N scores in one call) |
| `one_to_one: true` | One model call per hit. Inputs are scalar (single hit's fields). Output is per-hit. | **Per-hit enrichment when the model can't batch** — e.g., a custom classifier whose API only accepts one document at a time |

### Cross-document RAG (the most common use case)

**Connector** — single `prompt` parameter. The pipeline assembles the full prompt template, passes it to the connector as `parameters.prompt`:

```http
POST /_plugins/_ml/connectors/_create
{
  "name": "Claude RAG cross-doc",
  "version": "1.0",
  "protocol": "aws_sigv4",
  "credential": { "roleArn": "arn:aws:iam::<account>:role/<connector-role>" },
  "parameters": {
    "service_name": "bedrock", "region": "us-west-2",
    "anthropic_version": "bedrock-2023-05-31",
    "max_tokens": 400, "temperature": 0
  },
  "actions": [{
    "action_type": "predict", "method": "POST",
    "url": "https://bedrock-runtime.us-west-2.amazonaws.com/model/us.anthropic.claude-sonnet-4-6/invoke",
    "headers": { "content-type": "application/json" },
    "request_body": "{\"anthropic_version\":\"${parameters.anthropic_version}\",\"max_tokens\":${parameters.max_tokens},\"temperature\":${parameters.temperature},\"messages\":[{\"role\":\"user\",\"content\":\"${parameters.prompt}\"}]}"
  }]
}
```

**Pipeline** — `model_config.prompt` builds the full prompt with `${parameters.context.toString()}`:

```http
PUT /_search/pipeline/rag-pipeline
{
  "response_processors": [{
    "ml_inference": {
      "model_id": "<claude_model>",
      "function_name": "remote",
      "input_map": [{
        "context":  "title",
        "question": "$._request.query.match.title.query"
      }],
      "output_map": [{ "rag_answer": "content[0].text" }],
      "model_config": {
        "prompt": "You are a search assistant. Answer the user's question using ONLY the provided context list. Each list element is one retrieved document. If none of the documents contain the answer, reply: I do not know based on the retrieved documents. Cite which document(s) support your answer. Keep the answer under 80 words.\n\nContext: ${parameters.context.toString()}\n\nQuestion: ${parameters.question.toString()}"
      }
    }
  }]
}
```

**Searching:**

```http
POST /<index>/_search?search_pipeline=rag-pipeline
{
  "size": 3,
  "_source": ["title", "rag_answer"],
  "query": { "match": { "title": { "query": "What is the largest volcano?" } } }
}
```

Every hit's `_source` carries the **same** `rag_answer` (one model call, output replicated across hits). Application reads it from any hit and renders the standard "answer + supporting hits" UX.

### Critical gotchas

1. **Use `${parameters.X.toString()}` — not `${parameters.X}` — when the input is a list.** When `one_to_one: false`, the processor builds a `List<String>` for each `input_map` key. Without `.toString()`, gson serializes the list as a JSON-array literal `["doc1","doc2"]` whose internal double quotes break the surrounding JSON string in the connector's `request_body`. Apache Commons StringSubstitutor's `.toString()` method call converts the list to Java's plain `[doc1, doc2]` representation (no quotes), which is safe to embed inside a string slot. This is the difference between RAG that works and RAG that returns 400/Forbidden.

2. **`$._request.<jsonpath>` is the canonical way to read the original request body inside a response processor.** AOSS NextGen does not register `ext.ml_inference`, so the older `ext.ml_inference.<key>` extension pattern fails on AOSS. Use `$._request.query.match.title.query` to pull the user's question from the original search request.

3. **`output_map` writes to `_source` (per-hit), not response-level `ext`.** On open-source ml-commons, output_map keys starting with `ext.ml_inference.` get redirected to a top-level `MLInferenceSearchResponse.params` extension — this avoids cross-hit duplication. On AOSS NextGen, that branch is missing: an `ext.ml_inference.X` key falls through and writes to `_source` under a literal dotted field name. Always assume the output goes per-hit; for RAG, reading from any one hit is the application pattern.

4. **`one_to_one: true` is NOT real cross-doc RAG, and usually NOT the right rerank shape either.** Each `one_to_one: true` call sees only one document, so RAG synthesis can't compare evidence and most calls return "I do not know"; rerank with `one_to_one: true` makes one API call per hit (50× the latency and cost vs. a batched rerank API). Default to `one_to_one: false` for both synthesis (string output replicated to every hit) and batch scoring (array output fanned out via `results[*]`). Reach for `one_to_one: true` only when the underlying model API genuinely cannot batch.

### Per-hit re-ranking (Cohere Rerank — `one_to_one: false` with batch fan-out)

Use this when the model produces a relevance score per (query, document) pair, and you want to reorder hits by that score. Cohere Rerank is the canonical hosted rerank endpoint and accepts the full document list in one call. The `rerank` response processor (separate from `ml_inference`) consumes the per-hit score and reorders results.

This pattern uses **`one_to_one: false`** and aggregates all hits' fields into a single batched call — same API-call shape as cross-doc RAG, but the model returns a per-hit array that gets fanned back out using the `results[*]` index pattern in `output_map`. This matches the upstream [Cohere Rerank tutorial](https://github.com/opensearch-project/ml-commons/blob/main/docs/tutorials/ml_inference/rerank/ml_Inference_with_Cohere_Rerank_model.md).

**Connector:**

```http
POST /_plugins/_ml/connectors/_create
{
  "name": "Cohere Rerank",
  "version": "1.0",
  "protocol": "http",
  "credential": { "cohere_key": "<your-cohere-api-key>" },
  "parameters": {
    "model": "rerank-english-v3.0",
    "return_documents": true
  },
  "actions": [{
    "action_type": "predict",
    "method": "POST",
    "url": "https://api.cohere.ai/v1/rerank",
    "headers": { "Authorization": "Bearer ${credential.cohere_key}" },
    "request_body": "{ \"documents\": ${parameters.documents}, \"query\": \"${parameters.query}\", \"model\": \"${parameters.model}\", \"top_n\": ${parameters.top_n}, \"return_documents\": ${parameters.return_documents} }"
  }]
}
```

`${parameters.documents}` substitutes outside any quotes — Cohere's API expects a JSON array there directly. When the response processor aggregates hits into a `List<String>`, gson serializes it as `["doc1","doc2",...]` which is valid JSON in this position.

**Pipeline:**

```http
PUT /_search/pipeline/rerank-pipeline
{
  "response_processors": [
    {
      "ml_inference": {
        "tag": "cohere_rerank",
        "model_id": "<cohere_rerank_model>",
        "function_name": "remote",
        "one_to_one": false,
        "input_map": [{
          "documents": "fact_description",
          "query":     "_request.ext.query_context.query_text",
          "top_n":     "_request.ext.query_context.top_n"
        }],
        "output_map": [{
          "relevance_score": "results[*].relevance_score"
        }]
      }
    },
    {
      "rerank": {
        "by_field": {
          "target_field": "relevance_score",
          "remove_target_field": false,
          "keep_previous_score": false
        }
      }
    }
  ]
}
```

Three things to notice:

- **`one_to_one: false`** — the processor batches all hits into a single Cohere call. With 50 hits, you make 1 API call (not 50).
- **`results[*].relevance_score`** — the `[*]` wildcard tells the processor "the model returns an aligned array; fan element `i` back out to hit `i`." Each hit ends up with its own `_source.relevance_score`.
- **`_request.ext.query_context.query_text`** — `_request.*` (no `$.` prefix in this style; both forms work) reads from the original search request body, including the `ext` block. This is how you pass the rerank query.

**Searching:**

```http
POST /<index>/_search?search_pipeline=rerank-pipeline
{
  "size": 5,
  "query": { "match_all": {} },
  "ext": {
    "query_context": {
      "query_text": "Where do people go to see a show?",
      "top_n": "10"
    }
  }
}
```

The `query` does the broad retrieval (here `match_all`; in practice usually a `match` or `neural` query for the candidate pool). The response processor reranks each hit against the user's `query_text` and rewrites the order via the `rerank` processor. Reranked top-N is what the user sees.

**Note on AOSS NextGen:** the `rerank` response processor is in the curated allowlist (works), distinct from `collapse` which is not (returns `400`). See Section 4 for the full allowlist.

### How to debug an opaque "Forbidden" on AOSS

When AOSS returns `403 Forbidden` from a search pipeline call, the underlying status is masked (any non-429 4xx is rewritten to 403 — see `MachineLearningRestClient.handleResponseException` in the AOSS fork). Follow this chain:

1. **Confirm the model itself works.** Call `_predict` directly on the same model with the same parameters the pipeline would pass. If `_predict` succeeds, the model + IAM are good — the issue is in the pipeline.
2. **Run with `verbose_pipeline=true`.** This returns 200 even when a processor fails, with the per-processor `error` and `output_data` in `processor_results[]`. The processor-level error often surfaces the real reason ("Some parameter placeholder not filled in payload: text", "cannot find field", etc.) that the top-level 403 hid.
3. **Check input_map / connector parameter alignment.** The most common cause of opaque 403 is an `input_map` key that doesn't match a `${parameters.X}` placeholder in the connector's `request_body`. Open-source surfaces this as a clear 400 with the missing parameter name; AOSS strips the message.
4. **Replay on open-source if possible.** A local OpenSearch 3.x cluster with the same connector body will return the original error message. Worth setting up if you'll do significant pipeline development.

### Symptom → cause cheat sheet

If you see one of these symptoms on AOSS NextGen, the cause is usually:

| Symptom | Likely cause | Fix |
|---|---|---|
| `403 Forbidden` with no detail on a search using a working pipeline | Underlying 4xx error from the model is being masked. Run with `verbose_pipeline=true` to see the per-processor error. | Read processor_results to find the real failure |
| RAG answer appears as a literal field named `_source["ext.ml_inference.rag_answer"]` (with the dot in the key) on every hit | AOSS fork of `MLInferenceSearchResponseProcessor` lacks the `EXTENSION_PREFIX` write path. Output keys starting with `ext.ml_inference.` fall through to per-hit `_source` instead of the response-level `ext` block. | Rename the `output_map` key to a plain identifier like `rag_answer`. Read it from any one hit. |
| Rewritten query has `${input_map.X}` left as a literal string | The placeholder was used in the wrong field. `${input_map.X}` substitutes only inside `model_input`. `query_template` (request processor) substitutes only `output_map` keys; `model_config.prompt` (response processor) substitutes `${parameters.X}` and `${parameters.X.toString()}`. | If you wrote `${input_map.X}` in a `query_template`, switch to `${parameters.X}` and remember the input_map key must match a connector parameter. AOSS NextGen also doesn't expose `model_input`, so the workaround on AOSS is to bake the prompt into the connector's `request_body` and reference values via `${parameters.X}`. See Section 1. |
| `Invalid processor type collapse` when creating a search pipeline | The `collapse` response processor isn't in the AOSS NextGen allowlist. | Use `rerank` (which is allowlisted) or post-process client-side. |
| Bulk RAG response duplicates the same answer on every hit | This is by design when `output_map` writes to a `_source` field with `one_to_one: false` — one answer is replicated across all hits. Read it from any one hit; render the rest as supporting sources (Perplexity / Google AI Overviews UX pattern). | Not a bug — read from any hit |

---

## 4. AOSS NextGen specifics

The AOSS fork of `MLInferenceSearchRequestProcessor` and `MLInferenceSearchResponseProcessor` has been trimmed compared to upstream open-source ml-commons. When the user is on AOSS NextGen, apply these constraints:

| Feature | Open-source | AOSS NextGen |
|---|---|---|
| `ml_inference` ingest, request, response processors | ✅ | ✅ (NextGen only — Classic does not support `ml_inference`) |
| `model_input` field on processors | ✅ | ❌ Not supported. Pass model parameters via the connector's `${parameters.X}` template instead |
| `ext.ml_inference.X` write to response-level extension | ✅ | ❌ Falls through to per-hit `_source` (silent fallback) |
| `$._request.<jsonpath>` reads from request body | ✅ | ✅ |
| `$.X` JSONPath in output_map (per-hit `_source` write) | ✅ | ✅ |
| `query.X` JSONPath in output_map (rewrite existing slot) | ✅ | ✅ — but the path must already exist in the request body |
| `ext.ml_inference` SearchExtBuilder for passing params via the request `ext` block | ✅ | ❌ Not registered on AOSS NextGen (typically surfaces as a `NamedWriteable not found` error). Use `$._request.<jsonpath>` syntax to read from the request body instead. |
| `set`, `script`, `remove` ingest processors | ✅ | ❌ Curated allowlist — only `ml_inference`, `text_embedding`, and other ML-specific processors are exposed |
| `collapse` response processor | ✅ | ❌ Returns `400: Invalid processor type collapse` |
| Bedrock cross-region inference profiles (`us.amazon.nova-micro-v1:0`, `us.anthropic.claude-...`) | ✅ | ✅ |
| Generic 4xx errors from `_predict` are remapped to `403 Forbidden` | (n/a) | ⚠️ Yes — for any non-429 4xx, AOSS strips the original status and message. Use `verbose_pipeline=true` to see the per-processor error (which often surfaces the real reason). For deeper diagnosis, reproduce on open-source 3.7+ to see the unmasked error. |

When the user reports an opaque "Forbidden" on AOSS, your first move is `verbose_pipeline=true` — the processor-level error there often shows the real failure (e.g. `"Some parameter placeholder not filled in payload: text"`).

---

## 5. Choosing models

Pick the smallest model that handles the task. Different processors can use different models in the same pipeline.

| Task | Recommended Bedrock model | Why |
|---|---|---|
| Text embedding (NL → kNN) | `amazon.titan-embed-text-v2:0` (1024 dim, on-demand) | Fast, on-demand FM, well-supported by built-in pre/post-process functions |
| Language detection, intent classification, spell-correct | `us.amazon.nova-micro-v1:0` (cross-region inference profile) | Purpose-built for low-latency text classification; sub-second median latency |
| Per-hit re-ranking | `us.amazon.nova-micro-v1:0` or any small classifier | Per-hit calls accumulate latency, so smaller is better |
| RAG answer synthesis | `us.anthropic.claude-sonnet-4-6` or `us.anthropic.claude-haiku-4-5-20251001-v1:0` | Reasoning-quality matters more than latency for synthesis |
| Multi-step RAG with citations | Claude Sonnet 4.6 | Best instruction-following for grounded synthesis prompts |

For non-Bedrock connectors (SageMaker, OpenAI, self-hosted), the same shape applies — only the connector's `url` and `request_body` change. The pipeline itself is connector-agnostic.

---

## 6. Verification workflow

For every pipeline you build, verify in this order:

1. **Direct `_predict` first.** Call the model directly to confirm the connector + IAM are good before adding any pipeline complexity:
   ```bash
   POST /_plugins/_ml/models/<model_id>/_predict
   { "parameters": { "text": "Bonjour le monde" } }
   ```
2. **Pipeline `_simulate`** (ingest only — request/response pipelines do not have a simulate endpoint).
3. **`verbose_pipeline=true`** on the actual search to see each processor's input and output:
   ```http
   POST /<index>/_search?search_pipeline=<name>&verbose_pipeline=true
   { ... }
   ```
   The `processor_results` array shows each processor's `status`, `error`, `input_data`, and `output_data`. This is the single most useful debugging tool — always reach for it before assuming a permission or model issue.
4. **Compare against open-source if you suspect AOSS error masking.** If you see opaque `"Forbidden"` on AOSS and have access to a local OpenSearch 3.x cluster with the same connector, replay the request there to see the unmasked error message.

---

## 7. Decision flow

When a user asks for query-time or response-time LLM enrichment:

1. **What stage?**
   - Before retrieval, modify the query → request processor
   - After retrieval, enrich the response → response processor
   - At index time → use ingestion skill, not this one

2. **Single output for the whole request, or per-hit?**
   - Whole request (RAG synthesis, response-level summary) → `one_to_one: false` + `$._request.*` to read the question. Output is replicated across hits.
   - Per hit, model can batch (most LLM rerank APIs, Cohere Rerank) → `one_to_one: false` + `results[*]` in `output_map` to fan out the array. One API call, per-hit scores.
   - Per hit, model can't batch (rare; custom classifier with single-doc API) → `one_to_one: true`. N API calls.

3. **Does the model output drive the entire query, or just a slot?**
   - Entire query (NL → kNN where the vector IS the query) → `query_template` (Pattern A in Section 2)
   - Just a slot (add a filter to user's existing query) → `output_map` JsonPath rewrite (Pattern B in Section 2)

4. **Are we on AOSS NextGen?**
   - Yes → review Section 4 constraints. No `model_input`, no `ext.ml_inference.*`, no `collapse`. Read errors with `verbose_pipeline=true`.
   - No (managed OS or self-hosted) → all upstream features available.

5. **Pick the smallest model that does the job** — see Section 5.
