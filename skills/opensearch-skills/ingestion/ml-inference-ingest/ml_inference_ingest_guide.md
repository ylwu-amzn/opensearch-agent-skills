# ML Inference Ingest Processor Guide

This guide covers the `ml_inference` **ingest** processor — the variant that runs on every document at index time. For the search-time variants (request and response processors), see [`ml_inference_search_pipeline_guide.md`](../../search/opensearch-launchpad/ml_inference_search_pipeline_guide.md).

---

## 1. Why `ml_inference` instead of the specialized processors

OpenSearch ships a handful of specialized ingest processors for common ML tasks:

| Specialized processor | What it does |
|---|---|
| `text_embedding` | Run a registered embedding model on a text field, write the vector to a target field |
| `sparse_encoding` | Same shape but for sparse-encoding models (neural sparse search) |
| `text_image_embedding` | Multi-modal embedding |

Use these when the task is one they're built for — they're simpler. The specialized processor's whole config is `model_id` + `field_map`, no template plumbing.

Use `ml_inference` instead when:
- The model is **not** an embedding/encoding model — anything that returns text (classification, NER, summarization, redaction, free-form generation)
- You want fine control over the model invocation (non-trivial prompts, multi-field input, structured output extraction)
- You want to chain multiple enrichments in one pipeline (language detection → sentiment → NER, all calling different models)

`ml_inference` is the generic Swiss Army knife. It can call any registered model, with any prompt shape, and write any JSONPath of the response into any document field.

---

## 2. The three substitution namespaces (same as search variants)

| Namespace | Where it appears | Source of values |
|---|---|---|
| `${parameters.X}` | Inside the **connector's** `request_body` template | Connector's static `parameters` + per-doc parameters from `input_map` |
| `${input_map.X}` | Inside the processor's `model_input` field (open-source) — **not supported on AOSS NextGen** | The JSONPath value extracted by `input_map[X]` |
| `${X}` (where `X` is an `output_map` key) | Generally not used in ingest pipelines | (Not applicable here — ingest path doesn't use `query_template`) |

**The most common ingest pattern** uses only the first namespace: pipeline `input_map` populates parameters with field values from the document, the connector's `request_body` references them via `${parameters.X}`. No `model_input`, no `query_template`.

---

## 3. The canonical ingest pattern

### Connector

The connector's `request_body` references `${parameters.X}` for whatever fields the pipeline will pass:

```http
POST /_plugins/_ml/connectors/_create
{
  "name": "Nova Micro classifier",
  "version": "1.0",
  "protocol": "aws_sigv4",
  "credential": { "roleArn": "arn:aws:iam::<account>:role/<role>" },
  "parameters": {
    "service_name": "bedrock", "region": "us-west-2",
    "max_new_tokens": 10, "temperature": 0
  },
  "actions": [{
    "action_type": "predict", "method": "POST",
    "url": "https://bedrock-runtime.us-west-2.amazonaws.com/model/us.amazon.nova-micro-v1:0/invoke",
    "headers": { "content-type": "application/json" },
    "request_body": "{\"system\":[{\"text\":\"You classify text sentiment. Reply ONLY with one of: positive, negative, neutral. No explanation.\"}],\"messages\":[{\"role\":\"user\",\"content\":[{\"text\":\"${parameters.text}\"}]}],\"inferenceConfig\":{\"maxTokens\":${parameters.max_new_tokens},\"temperature\":${parameters.temperature}}}"
  }]
}
```

### Model registration

```http
POST /_plugins/_ml/models/_register
{
  "name": "Nova Micro sentiment",
  "function_name": "remote",
  "description": "Sentiment classifier",
  "connector_id": "<from connector creation>"
}

POST /_plugins/_ml/models/<model_id>/_deploy
```

### Ingest pipeline

```http
PUT /_ingest/pipeline/sentiment-pipeline
{
  "description": "Auto-tag every doc with sentiment",
  "processors": [{
    "ml_inference": {
      "model_id": "<model_id>",
      "input_map":  [{ "text": "review_text" }],
      "output_map": [{ "sentiment": "output.message.content[0].text" }]
    }
  }]
}
```

**`input_map`**: `{ "<connector param name>": "<document field name>" }`. The processor extracts the document's `review_text` field and binds it to `parameters.text`, which the connector's `${parameters.text}` placeholder consumes.

**`output_map`**: `{ "<target field in _source>": "<jsonpath into model response>" }`. The model returns its prediction at `output.message.content[0].text` (Nova format); the processor reads that value and writes it to `_source.sentiment`.

### Index with the pipeline as default

```http
PUT /reviews
{
  "settings": { "default_pipeline": "sentiment-pipeline" },
  "mappings": {
    "properties": {
      "review_text": { "type": "text" },
      "sentiment":   { "type": "keyword" }
    }
  }
}
```

Now every document indexed into `reviews` automatically gets a `sentiment` field.

### Verification

Always `_simulate` before bulk-loading real data:

```http
POST /_ingest/pipeline/sentiment-pipeline/_simulate
{
  "docs": [
    { "_index": "reviews", "_source": { "review_text": "I love this product, it changed my life!" } },
    { "_index": "reviews", "_source": { "review_text": "Worst purchase I've ever made." } }
  ]
}
```

Expected: each simulated doc's `_source` has the original `review_text` plus a new `sentiment: "positive"` or `"negative"`. If not, fix `output_map` JSONPath before any real ingest.

---

## 4. Chained enrichment (multiple models in one pipeline)

To enrich documents with multiple independent signals (e.g., language + sentiment + NER), chain `ml_inference` processors in one pipeline. Each processor calls its own model and writes its own field.

The example below assumes all three models are **Nova Micro** (purpose-built for low-latency text classification). If you're using a different model family, the `output_map` JSONPath has to match that model's response shape — see Section 6 gotcha 2 for the per-family path table.

```http
PUT /_ingest/pipeline/multi-nlp-pipeline
{
  "description": "Language + sentiment + entities for every doc (Nova Micro)",
  "processors": [
    {
      "ml_inference": {
        "model_id": "<nova_lang_detect_model>",
        "input_map":  [{ "text": "text" }],
        "output_map": [{ "language": "output.message.content[0].text" }]
      }
    },
    {
      "ml_inference": {
        "model_id": "<nova_sentiment_model>",
        "input_map":  [{ "text": "text" }],
        "output_map": [{ "sentiment": "output.message.content[0].text" }]
      }
    },
    {
      "ml_inference": {
        "model_id": "<claude_ner_model>",
        "input_map":  [{ "text": "text" }],
        "output_map": [{ "entities_raw": "$.content[0].text" }]
      }
    }
  ]
}
```

Notice the third processor uses Claude (for the more nuanced NER task) — its `output_map` path is `$.content[0].text` (Claude shape), not `output.message.content[0].text` (Nova shape). Different models in the same chained pipeline need different output paths. This is normal and expected.

Each processor uses its own task-specific model (with its own system prompt baked into the connector). The pipeline runs them in order on each document; later processors can read fields that earlier processors wrote.

**Three task-specific models**, not one model with three prompts: the prompt lives in the connector's `request_body` (often via the `system` field), so each task gets its own connector. This makes prompts versioned and reusable. See [connector_patterns.md](connector_patterns.md) for ready-to-use connector recipes per task.

---

## 5. AOSS NextGen specifics

| Constraint | Detail |
|---|---|
| `ml_inference` ingest processor support | NextGen only. Classic AOSS collections do not support it. |
| `model_input` field | **Not supported.** Pass model parameters via the connector's `${parameters.X}` template instead. |
| `set`, `script`, `remove` ingest processors | **Not in the allowlist.** AOSS NextGen exposes only ML-related processors (`ml_inference`, `text_embedding`, `sparse_encoding`, etc.). To dynamically build a prompt per document, embed the prompt structure in the connector's static `request_body` and only pass field values via `input_map`. |
| Bedrock cross-region inference profiles (`us.amazon.nova-micro-v1:0`, `us.anthropic.claude-...`) | ✅ Fully supported. The `ml_inference` processor invokes inference profiles the same way as on-demand foundation models — no special config. |
| 4xx error remapping to 403 Forbidden | ⚠️ When a `_predict` call returns any non-429 4xx, AOSS strips the original status and message and returns a generic `403 Forbidden`. Use `_simulate` (which surfaces processor-level errors better) or reproduce on open-source 3.x to see the unmasked error. |
| `aoss:CreateMLResource` / `ExecuteMLResource` permissions | Required in the data access policy. Without them you'll get `Authorization of request failed` on `_create` / `_predict`. |

When the user is on AOSS, also confirm the connector role's trust policy allows `ml.opensearchservice.amazonaws.com` (or `opensearchservice.amazonaws.com` for the broader pattern) and the role has `bedrock:InvokeModel` on the specific model ARN or inference profile ARN.

---

## 6. Common gotchas

1. **Connector parameter name must match `input_map` key.** If `request_body` references `${parameters.text}`, your `input_map` must have `text` as a key. A mismatch on AOSS surfaces as opaque `403 Forbidden`; on open-source it's `parameter placeholder not filled in payload: text`.

2. **Field values are substituted into `request_body` as raw strings — unescaped.** The `${parameters.text}` placeholder is replaced by the literal field value before the body is parsed as JSON. If a document's field contains a double-quote, backslash, or newline (very common for real reviews, articles, or PII-laden text), the interpolated `request_body` becomes invalid JSON and the call fails — on AOSS as an opaque `403 Forbidden`, on open-source as `Invalid payload`. This is the single most common real-world failure for these recipes. Mitigations:
   - Note that moving the value into a content-block array (`"content":[{"type":"text","text":"${parameters.text}"}]` instead of `"content":"${parameters.text}"`) does **not** help — the substitution is still raw, so a quote or newline in the value breaks the JSON either way.
   - **Rely on a `pre_process_function`** (the embedding recipes in [connector_patterns.md](connector_patterns.md) Section 7 use one) which builds the request body programmatically with proper escaping, instead of raw template substitution.
   - **Always `_simulate` with a document that contains quotes and newlines**, not just clean sample text — clean samples hide this bug until production data hits the pipeline.

3. **`output_map` JSONPath must match the model's actual response shape.** Different model families return text at different paths:
   - Bedrock Titan embeddings: `$.embedding`
   - Bedrock Claude (Anthropic Messages API): `$.content[0].text` (or `content[0].text` in search response processors — see note below). Note the asymmetry: the request may send `content` as a bare string, but the **response** `content` is always an array of content blocks, so you read `content[0].text`.
   - Bedrock Nova / Bedrock Converse API: `output.message.content[0].text`
   - Bedrock Llama (3.x and 4.x): `generation`
   - SageMaker / OpenAI / others: depends on the deployment; inspect `_predict` output first

   **Always run direct `_predict` first** to see the response shape, then write `output_map` to match.

   **`$.` prefix differs by processor type.** In **ingest** processors, the JSONPath needs the `$.` prefix to root at the model's full inference response (e.g., `$.content[0].text`). In **search response** processors, the JSONPath roots into the model output already, so the `$.` is omitted (e.g., `content[0].text`). This is because `full_response_path` defaults differently for the two processor types. If you copy-paste from a search-response example into an ingest pipeline (or vice versa), the path will be wrong by exactly the `$.` prefix.

4. **The pipeline runs synchronously per document.** A slow model (e.g., a 5s LLM call) will dominate ingest throughput. For high-volume ingestion with LLM enrichment, either (a) batch ingestion in off-peak windows, (b) use a small fast classifier (Nova Micro is purpose-built for this), or (c) run enrichment as a separate offline job and reindex.

5. **Ingest pipeline errors fail the whole bulk operation.** Set `"ignore_failure": true` per processor if a single doc's enrichment failure shouldn't poison the whole batch — but be careful: silent failures are harder to debug than loud ones.

6. **Field types in the index mapping must match what the pipeline writes.** If `output_map` writes a string but the index mapping says `keyword`, fine. If the mapping says `knn_vector` but the model returns a string, you'll get a mapper exception per doc. Get the mapping right before bulk-loading.

7. **The pipeline can read fields that earlier processors wrote.** In a chained pipeline, the second processor's `input_map` can reference a field that the first processor's `output_map` produced. This enables conditional enrichment (e.g., "first detect language, then route to a language-specific model") — though for simple cases, one model with multilingual support is usually cleaner.

---

## 7. Verification commands

```bash
# Health
uv run python scripts/opensearch_ops.py status

# Direct predict (sanity check the connector + model)
curl -sk -u admin:<password> -X POST https://localhost:9200/_plugins/_ml/models/<model_id>/_predict \
  -H 'Content-Type: application/json' \
  -d '{"parameters": {"text": "I love this product"}}'

# _simulate (sanity check the pipeline before any real ingest)
curl -sk -u admin:<password> -X POST https://localhost:9200/_ingest/pipeline/<name>/_simulate \
  -H 'Content-Type: application/json' \
  -d '{"docs":[{"_index":"test","_source":{"text":"sample"}}]}'

# Inspect a real ingested doc's _source
uv run python scripts/opensearch_ops.py search --index <name> --body '{"size":1,"_source":true}'
```

---

## 8. Decision flow

When the user wants document enrichment at index time:

1. **Is the task an embedding?** → Use the specialized `text_embedding` processor instead. Simpler, fewer config errors. Use `ml_inference` only if you need an embedding model the specialized processor doesn't support natively.
2. **One model or multiple?** → One model: single `ml_inference` processor. Multiple independent enrichments: chained pipeline (Section 4). Multiple dependent enrichments (later one needs earlier output): chained, with `input_map` of later processors reading the field the earlier one wrote.
3. **What model?** → Read [connector_patterns.md](connector_patterns.md). For text classification / extraction, `us.amazon.nova-micro-v1:0` is purpose-built (low latency, low cost). For free-form generation (summarization, complex extraction), Claude Sonnet 4.6 or Haiku 4.5.
4. **Verify in this order**: direct `_predict` → `_simulate` → small bulk → full ingest. Don't skip steps.
5. **AOSS or self-hosted?** → If AOSS NextGen, review Section 5. Most importantly, AOSS masks 4xx errors as 403 — use `_simulate` and (if available) compare against open-source to debug.
