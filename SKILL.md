---
name: rag-engineer
description: "Use when building, debugging, reviewing, or extending a retrieval-augmented generation, document-QA, semantic-search, vector-search, embedding, chunking, citation, reranking, or hallucination/refusal system. Trigger when answers are missing, wrong, uncited, unauthorized, slow, or degraded by an index/backend failure."
---

# RAG Engineer

Retrieval quality bounds generation quality. A better prompt cannot recover a fact that was never indexed, retrieved, authorized, or preserved in the final context.

## Scope and routing

This file is the portable, company-neutral core. When building a RAG application for a new company or tenant, read `references/multi-company-architecture.md` before choosing indexes, schemas, or access boundaries.

## Diagnose in pipeline order

```text
source → parse/chunk → embed/index → authorize/filter → retrieve
       → deduplicate/rerank → assemble evidence → generate → cite → evaluate
```

When an answer is wrong, check:

1. Was the source parsed, chunked, and indexed?
2. Did the expected source survive pre-retrieval authorization and appear in raw results?
3. Did it survive deduplication and context budgeting?
4. Does each citation identify evidence actually shown to the model?
5. Did generation use that evidence faithfully?
6. Which evaluation metric proves the diagnosis?
7. Is the result safe, observable, affordable, and fast enough to ship?

Do not use a prompt change to fix a retrieval, authorization, citation, or backend-outage problem.

## Minimum safe architecture

Start with a parser, structure-aware chunks, canonical document/chunk identities, vector retrieval with server-side access filtering, explicit context budgeting, grounded generation, stable citations, and deterministic evaluation. Add keyword search for identifiers, reranking for ordering, and graph/summary/vision indexes only when an evaluation slice shows a measurable need. Every backend must report status separately: an empty result caused by an outage is not the same as a healthy empty result.

## Quick decisions

| Need | First option |
|---|---|
| Meaning/paraphrase | Vector search |
| IDs, acronyms, exact terms | Keyword/full-text search |
| Good recall, poor ordering | Rerank top candidates |
| Multi-hop evidence | Graph search, only if measured |
| Long or ambiguous documents | Summary/parent retrieval, only if measured |
| Access control | Server-resolved tenant/collection filters before retrieval |

Do not choose a vector database or chunk size from this table alone. Measure on the target corpus and question shapes.

## References

| File | Read when… |
|---|---|
| `references/multi-company-architecture.md` | Designing a reusable company/tenant RAG application |
| `references/chunking-strategies.md` | Choosing chunks or diagnosing missing facts |
| `references/embedding-models.md` | Choosing embeddings, vector stores, or refresh strategy |
| `references/retrieval-patterns.md` | Implementing hybrid retrieval, reranking, deduplication, or filters |
| `references/context-assembly-and-citations.md` | Evidence was retrieved but the answer is wrong or uncited |
| `references/evaluation.md` | Building an eval set or separating retrieval from generation failures |
| `references/production-reliability.md` | Reviewing security, latency, cost, observability, or outages |

## Adapting to your codebase

Once you're working inside an actual repository, add one more reference file — `references/<your-app>-implementation.md` — that maps these principles onto your real code: your actual entry points, retrieval backends, context-assembly function, citation logic, config variables, and known gotchas, each with file/line references you've verified yourself. Write it the way an onboarding engineer would want it written: concrete, checked against the source, and corrected the moment it drifts from the code. Never let a generic pattern file (this one, or any file in `references/`) accumulate project-specific paths, endpoints, or vendor names — that's what the adapter file is for.
