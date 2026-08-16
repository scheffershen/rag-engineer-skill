# Portable Multi-Company RAG Architecture

Use this reference when adapting a RAG system to a new company, tenant, or corpus. It is company-neutral.

## Minimum safe architecture

Start with the smallest system that can satisfy the measured query set:

1. Source connector and parser/OCR pipeline.
2. Structure-aware chunks with stable source metadata.
3. Vector retrieval with server-side tenant/collection filtering.
4. Context assembly with an explicit evidence budget.
5. Grounded generation with structured citations.
6. Deterministic retrieval and answer evaluation.

Add keyword search for identifiers and exact terms. Add reranking when first-stage recall is good but ordering is poor. Add summaries, graph search, vision, or query rewriting only when an evaluation slice shows a measurable need. More backends increase cost, latency, failure modes, and evidence-duplication risk.

## Non-negotiable data contracts

Every indexed chunk should carry, or be joinable to:

```text
tenant_id / collection_id
document_id
document_version_id
chunk_id
source_uri or stable source key
page / section / heading when available
access policy or ACL reference
content_hash
parser_version
embedding_model and embedding_version
```

Use the same canonical document and chunk identity across vector, keyword, graph, summary, and evaluation stores. Backend-specific IDs may exist, but they must map back to the canonical identity. Re-indexing, model changes, and deletion must be idempotent and observable.

## Authorization boundary

The safe query sequence is:

```text
authenticated principal
  → server-resolved allowed collections/documents
  → pre-retrieval filters
  → retrieval
  → context assembly
  → generation
```

Never treat a request-supplied `workspace`, `tenant_id`, collection name, label, or prompt instruction as proof of authorization. Never retrieve broadly and remove unauthorized text afterward. Add a negative test proving an ineligible document is absent from raw retrieval results for every role or boundary.

## Query flow contract

Run enabled retrieval branches in parallel where their clients permit it. Each branch should return results plus status, latency, and degradation information. Merge by canonical chunk/document identity, deduplicate before spending the context budget, optionally rerank, assemble evidence, generate, and validate citations against the evidence actually shown.

An empty result from a failed backend is not equivalent to a healthy empty result. Preserve a `degraded` or backend-status signal so the API and telemetry can distinguish "no evidence found" from "one evidence source was unavailable."

## Ingestion and lifecycle

Treat ingestion as a state machine rather than a single successful upload:

```text
discovered → parsed → chunked → indexed per backend → verified → active
                                      ↘ failed/retryable
```

Track per-backend status. Re-index changed content using a content hash and version. On deletion, remove source chunks and all derived records that are no longer supported by another document; do not leave graph entities, summaries, or keyword records orphaned without an explicit retention decision.

## Evaluation gates

Before a new-company pilot, require:

- metadata-only retrieval recall/union recall by question shape;
- source survival after deduplication and context budgeting;
- citation identity and citation fidelity;
- strict answer accuracy and refusal accuracy;
- negative authorization tests at the raw-retrieval boundary;
- re-index, deletion, and embedding-version consistency tests;
- backend failure/degraded-response tests;
- p50/p95 latency and per-request cost by backend.

RAGAS or another LLM judge can supplement these checks. It must not replace deterministic retrieval, authorization, lifecycle, and citation tests.
