---
name: ingestion
description: >
  Enrich and transform documents at index time in OpenSearch using ingest
  pipelines. Use this skill when the user mentions ingest pipeline, ingest
  processor, document enrichment at index time, auto-embedding, automatic
  classification, sentiment tagging, language detection at ingest, named
  entity recognition (NER), PII redaction, ML inference processor (ingest
  variant), text_embedding processor, or any related document-enrichment
  topic that runs as documents flow into the index.
compatibility: Requires Docker and uv. Connector-based enrichment (Bedrock, SageMaker, OpenAI) requires the corresponding model credentials.
metadata:
  author: opensearch-project
  version: "1.0"
---

# Ingestion

Category skill for enriching and transforming documents at index time with OpenSearch ingest pipelines.

## Skills

| Skill | Description |
|---|---|
| [ml-inference-ingest](ml-inference-ingest/SKILL.md) | Wire any ML model (embeddings, LLMs, classifiers) into an OpenSearch ingest pipeline so every document gets enriched automatically — auto-embeddings for vector search, language tags, sentiment, NER, PII redaction, summarization at index time |

## When to Use

Read [ml-inference-ingest/SKILL.md](ml-inference-ingest/SKILL.md) when the user wants to:

- Generate embeddings automatically for every ingested document (auto-vectorization for kNN/semantic search)
- Auto-tag documents with language, sentiment, intent, or topic at index time
- Extract named entities (PERSON, ORG, LOCATION, etc.) from each document
- Redact PII from documents before they hit storage
- Summarize long documents into a short field for filtering or display
- Classify documents into categories using an LLM
- Run any other document transformation that calls a remote ML model

## Related Skills

- For **query-time** LLM enrichment (query rewriting, RAG answer synthesis), use [search/opensearch-launchpad](../search/opensearch-launchpad/SKILL.md) — specifically [ml_inference_search_pipeline_guide.md](../search/opensearch-launchpad/ml_inference_search_pipeline_guide.md). The processor is the same (`ml_inference`) but configured as a search request/response processor instead of an ingest processor.
- For end-to-end search application building (which usually combines ingest enrichment + a search strategy), start with [search/opensearch-launchpad](../search/opensearch-launchpad/SKILL.md). It will pull this skill in when document-side enrichment is needed.
