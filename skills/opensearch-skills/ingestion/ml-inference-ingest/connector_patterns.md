# Connector Patterns

Copy-pasteable connector recipes for the most common `ml_inference` use cases. The connector is where most user errors happen — start from a known-working recipe and only customize the parts you need.

> All recipes use `protocol: aws_sigv4` for Bedrock. Replace `${region}`, `${connector_role_arn}`, and credential values with your own. For SageMaker/OpenAI/self-hosted, see Section 5.

---

## 1. Bedrock Titan Text Embeddings v2 (auto-embedding for kNN)

**Use for:** generating dense vectors for semantic / kNN search.

```json
{
  "name": "Bedrock Titan Embed v2",
  "description": "Titan v2 text embedding for semantic search",
  "version": "1.0",
  "protocol": "aws_sigv4",
  "credential": {
    "roleArn": "${connector_role_arn}"
  },
  "parameters": {
    "service_name": "bedrock",
    "region": "${region}",
    "model": "amazon.titan-embed-text-v2:0",
    "dimensions": 1024,
    "normalize": true
  },
  "actions": [{
    "action_type": "predict",
    "method": "POST",
    "url": "https://bedrock-runtime.${region}.amazonaws.com/model/amazon.titan-embed-text-v2:0/invoke",
    "headers": {
      "content-type": "application/json"
    },
    "request_body": "{ \"inputText\": \"${parameters.inputText}\", \"dimensions\": ${parameters.dimensions}, \"normalize\": ${parameters.normalize} }",
    "pre_process_function": "connector.pre_process.bedrock.embedding",
    "post_process_function": "connector.post_process.bedrock.embedding"
  }]
}
```

**Pipeline `input_map`:** `[{ "inputText": "<source field>" }]`
**Pipeline `output_map`:** `[{ "<vector field>": "$.embedding" }]`
**Index mapping:** `<vector field>` must be `knn_vector` with `dimension: 1024` (or whatever you set in `parameters.dimensions`).

> **Tip:** if you only want auto-embedding, prefer the specialized `text_embedding` ingest processor — it's simpler than `ml_inference` for this single-task case. Use `ml_inference` when you need an embedding model the specialized processor doesn't natively support, or when chaining with other enrichments.

---

## 2. Bedrock Nova Micro (text classification — sentiment, language, intent)

**Use for:** small, fast, low-cost text classification at ingest time. Nova Micro is purpose-built for this. Sub-second latency end-to-end.

```json
{
  "name": "Nova Micro classifier",
  "description": "Generic Nova Micro classifier; specialize via system prompt",
  "version": "1.0",
  "protocol": "aws_sigv4",
  "credential": {
    "roleArn": "${connector_role_arn}"
  },
  "parameters": {
    "service_name": "bedrock",
    "region": "${region}",
    "max_new_tokens": 10,
    "temperature": 0
  },
  "actions": [{
    "action_type": "predict",
    "method": "POST",
    "url": "https://bedrock-runtime.${region}.amazonaws.com/model/us.amazon.nova-micro-v1:0/invoke",
    "headers": { "content-type": "application/json" },
    "request_body": "{\"system\":[{\"text\":\"${SYSTEM_PROMPT}\"}],\"messages\":[{\"role\":\"user\",\"content\":[{\"text\":\"${parameters.text}\"}]}],\"inferenceConfig\":{\"maxTokens\":${parameters.max_new_tokens},\"temperature\":${parameters.temperature}}}"
  }]
}
```

Replace `${SYSTEM_PROMPT}` (literally, when creating the connector — not at runtime) with the task instruction. Common templates:

| Task | System prompt |
|---|---|
| Language detection | `You detect language. Reply ONLY with the ISO-639-1 two-letter code (en, es, fr, de, ja, zh, etc). No explanation.` |
| Sentiment | `You classify text sentiment. Reply ONLY with one word: positive, negative, or neutral. No explanation.` |
| Intent (e-commerce) | `You classify shopper intent. Reply ONLY with one of: browse, compare, purchase, support, return. No explanation.` |
| Toxicity | `You classify text toxicity. Reply ONLY with one word: safe, toxic. No explanation.` |
| Topic (news) | `You classify news topics. Reply ONLY with one of: politics, business, technology, sports, entertainment, health, science. No explanation.` |

**Pipeline `input_map`:** `[{ "text": "<source field>" }]`
**Pipeline `output_map`:** `[{ "<target keyword field>": "output.message.content[0].text" }]`
**Index mapping:** target field is typically `keyword` for filtering / aggs.

---

## 3. Bedrock Claude Sonnet / Haiku (free-form generation — NER, summarization, redaction)

**Use for:** any task that needs structured or free-form text generation — extracting entities as JSON, summarizing long docs, redacting PII, classifying with explanation.

```json
{
  "name": "Claude task",
  "description": "Claude for ${TASK_DESC}",
  "version": "1.0",
  "protocol": "aws_sigv4",
  "credential": {
    "roleArn": "${connector_role_arn}"
  },
  "parameters": {
    "service_name": "bedrock",
    "region": "${region}",
    "anthropic_version": "bedrock-2023-05-31",
    "max_tokens": 300,
    "temperature": 0
  },
  "actions": [{
    "action_type": "predict",
    "method": "POST",
    "url": "https://bedrock-runtime.${region}.amazonaws.com/model/us.anthropic.claude-haiku-4-5-20251001-v1:0/invoke",
    "headers": { "content-type": "application/json" },
    "request_body": "{\"anthropic_version\":\"${parameters.anthropic_version}\",\"max_tokens\":${parameters.max_tokens},\"temperature\":${parameters.temperature},\"system\":\"${SYSTEM_PROMPT}\",\"messages\":[{\"role\":\"user\",\"content\":\"${parameters.text}\"}]}"
  }]
}
```

Replace `${SYSTEM_PROMPT}` and pick a model URL:
- `us.anthropic.claude-haiku-4-5-20251001-v1:0` — fastest, cheapest Claude. Good for ingest at scale.
- `us.anthropic.claude-sonnet-4-6` — higher quality. Use for complex extraction or when grounding fidelity matters.

System prompt templates:

| Task | System prompt |
|---|---|
| NER (JSON output) | `You extract named entities. Reply ONLY with a JSON array of objects with fields text and type (PERSON, ORG, LOCATION, DATE, PRODUCT). No markdown fence, no explanation.` |
| Summarization (one sentence) | `You summarize text. Reply ONLY with a single sentence under 25 words capturing the main point. No explanation, no preamble.` |
| PII redaction | `You redact personally identifiable information. Reply ONLY with the original text where each PII span is replaced with [REDACTED:type], where type is one of NAME, EMAIL, PHONE, SSN, ADDRESS, CARD. No explanation.` |
| Topic + explanation | `You classify document topic and briefly explain. Reply ONLY with JSON: {"topic": "<topic>", "reason": "<one sentence>"}. No markdown fence.` |

**Pipeline `input_map`:** `[{ "text": "<source field>" }]`
**Pipeline `output_map`:** `[{ "<target field>": "content[0].text" }]`

> **JSON output handling:** If the model is returning JSON (NER, structured classification), the `output_map` writes the raw JSON string into the document. To make it queryable as nested objects, either (a) parse client-side, or (b) chain a downstream processor that expects a JSON-shaped string. AOSS NextGen does not currently expose a `json` ingest processor, so client-side parsing is usually the cleaner path.

---

## 4. Bedrock Nova Pro / Llama 4 (longer-context generation, complex reasoning)

For ingest workloads where Haiku/Nova Micro isn't enough quality but Sonnet is too slow/expensive:

```json
{
  "url": "https://bedrock-runtime.${region}.amazonaws.com/model/us.amazon.nova-pro-v1:0/invoke",
  "request_body": "{\"system\":[{\"text\":\"${SYSTEM_PROMPT}\"}],\"messages\":[{\"role\":\"user\",\"content\":[{\"text\":\"${parameters.text}\"}]}],\"inferenceConfig\":{\"maxTokens\":${parameters.max_new_tokens},\"temperature\":${parameters.temperature}}}"
}
```

Same Nova invoke API as Micro/Lite — only the model URL changes. Output path is `output.message.content[0].text`.

For Llama 4 Scout / Maverick:
```json
{
  "url": "https://bedrock-runtime.${region}.amazonaws.com/model/us.meta.llama4-scout-17b-instruct-v1:0/invoke"
}
```
Llama uses the older completions-style request body — see the [Bedrock model API docs](https://docs.aws.amazon.com/bedrock/latest/userguide/model-parameters-meta.html) for the exact `request_body` shape (`prompt` + `max_gen_len`).

---

## 5. Non-Bedrock connectors

The same `ml_inference` processor works with any HTTP endpoint that takes JSON. Only the connector's `url`, `request_body`, and credential block change.

### SageMaker endpoint

```json
{
  "protocol": "aws_sigv4",
  "credential": { "roleArn": "${sagemaker_invoke_role_arn}" },
  "parameters": {
    "service_name": "sagemaker",
    "region": "${region}"
  },
  "actions": [{
    "action_type": "predict",
    "method": "POST",
    "url": "https://runtime.sagemaker.${region}.amazonaws.com/endpoints/${endpoint_name}/invocations",
    "headers": { "content-type": "application/json" },
    "request_body": "<the body shape your SageMaker endpoint expects>"
  }]
}
```

The IAM role needs `sagemaker:InvokeEndpoint` on the specific endpoint ARN.

### OpenAI / generic HTTPS API

```json
{
  "protocol": "http",
  "credential": { "openai_api_key": "<your key>" },
  "parameters": { "model": "gpt-4o-mini", "max_tokens": 200 },
  "actions": [{
    "action_type": "predict",
    "method": "POST",
    "url": "https://api.openai.com/v1/chat/completions",
    "headers": {
      "Authorization": "Bearer ${credential.openai_api_key}",
      "content-type": "application/json"
    },
    "request_body": "{\"model\":\"${parameters.model}\",\"messages\":[{\"role\":\"user\",\"content\":\"${parameters.text}\"}],\"max_tokens\":${parameters.max_tokens}}"
  }]
}
```

Output path for OpenAI chat completions: `choices[0].message.content`.

---

## 6. IAM setup checklist (Bedrock connectors)

For every Bedrock connector running on AOSS or AOS:

1. **Create an IAM role** for the connector to assume. Trust policy:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow",
       "Principal": { "Service": ["ml.opensearchservice.amazonaws.com", "opensearchservice.amazonaws.com"] },
       "Action": "sts:AssumeRole"
     }]
   }
   ```
2. **Attach an inline policy** with `bedrock:InvokeModel` on the specific model or inference profile ARN(s):
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow",
       "Action": "bedrock:InvokeModel",
       "Resource": [
         "arn:aws:bedrock:${region}::foundation-model/amazon.titan-embed-text-v2:0",
         "arn:aws:bedrock:${region}:${account}:inference-profile/us.amazon.nova-micro-v1:0",
         "arn:aws:bedrock:${region}:${account}:inference-profile/us.anthropic.claude-haiku-4-5-20251001-v1:0"
       ]
     }]
   }
   ```
3. **Add the role to the data access policy** (AOSS only) on the collection where the connector will be used. The role needs `aoss:CreateMLResource` / `ExecuteMLResource` on the model resource type, plus the standard collection/index permissions.
4. **Grant `iam:PassRole`** to the human/role that creates the connector (so they can pass the connector role to OpenSearch).

If the connector returns `Authorization error during prediction` or AOSS returns opaque `403 Forbidden` on `_predict`, the most common causes are: (a) trust policy missing `ml.opensearchservice.amazonaws.com`, (b) invoke policy missing the specific inference-profile ARN (cross-region inference profiles need their inference-profile ARN, not the foundation-model ARN), (c) AOSS data access policy missing `aoss:ExecuteMLResource`.

---

## 7. Pre/post-process functions

For embedding models, OpenSearch ships built-in pre/post-process functions that handle the request/response shaping for you. These are NOT applied by default — you must reference them in the connector:

```json
"pre_process_function": "connector.pre_process.bedrock.embedding",
"post_process_function": "connector.post_process.bedrock.embedding"
```

Available built-ins (most common):
- `connector.pre_process.bedrock.embedding` / `connector.post_process.bedrock.embedding` — Bedrock Titan / Cohere embeddings
- `connector.pre_process.cohere.embedding` / `connector.post_process.cohere.embedding`
- `connector.pre_process.openai.embedding` / `connector.post_process.openai.embedding`

For text generation models (Claude, Nova, Llama for non-embedding tasks), there are no pre/post-process functions in the built-in set — you control the shape via the `request_body` template directly, and `output_map` extracts the field you want from the raw model response.

---

## 8. Quick "which connector" decision

| Task | Recommended connector |
|---|---|
| Auto-embedding for kNN | Section 1 (Titan v2) |
| Sentiment / language / intent / topic | Section 2 (Nova Micro) |
| NER, summarization, PII redaction | Section 3 (Claude Haiku) |
| Long-context summarization, complex extraction | Section 4 (Nova Pro or Sonnet) |
| Custom fine-tuned model | Section 5 (SageMaker) |
| Quick prototype with off-AWS API | Section 5 (OpenAI/generic) |
