# Retrieval Patterns: Hybrid Search, Query Handling, Reranking

## Table of contents
- [Hybrid search: combining semantic and keyword signals](#hybrid-search-combining-semantic-and-keyword-signals)
- [Query handling before you touch retrieval](#query-handling-before-you-touch-retrieval)
- [Reranking](#reranking)
- [Deduplication](#deduplication)
- [Metadata filtering](#metadata-filtering)
- [Diagnosing a retrieval failure](#diagnosing-a-retrieval-failure)
- [Anti-patterns](#anti-patterns)

## Hybrid search: combining semantic and keyword signals

Vector search and full-text/keyword search fail on different question shapes, so production RAG systems almost always run both and merge:

- **Vector search** — meaning, paraphrase, cross-lingual concepts.
- **Full-text/keyword search (BM25, Meilisearch, Elasticsearch, Postgres `tsvector`)** — exact identifiers, acronyms, names, numbers.

That is a measured option, not a mandatory starting topology. For a new company, begin with vector retrieval plus server-side metadata/ACL filtering; add full-text only when exact-term questions underperform. Every additional branch consumes latency and evidence budget.

**Reciprocal Rank Fusion (RRF)** is the standard, parameter-light way to merge two ranked lists without needing to tune relative score weights (which aren't even on the same scale between a cosine similarity and a BM25 score):

```python
def reciprocal_rank_fusion(result_lists, k=60):
    """result_lists: list of ranked-id-lists, one per retrieval method."""
    scores = {}
    for results in result_lists:
        for rank, doc_id in enumerate(results):
            scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank + 1)
    return sorted(scores, key=scores.get, reverse=True)
```

**Before building a fused/hybrid pipeline, measure each signal's contribution separately.** Run vector-only and keyword-only retrieval against the same eval set and record, per question, which signal found the expected source. This tells you whether you actually need fusion for a given query type, or whether one signal alone already covers it — fusing blindly can bury a strong keyword hit under noisier vector candidates if weighted wrong.

## Query handling before you touch retrieval

Before reaching for a bigger index or a reranker, check whether the *query itself* is the problem:

- **Identifier extraction** — if the user's question contains something that looks like a document ID/SOP number/acronym, extract it and search it literally (exact match / full-text) in addition to the semantic query. This alone fixes a large fraction of "exact lookup" failures.
- **Query normalization** — expand abbreviations, translate cross-lingual terms, strip conversational filler ("can you tell me...") that dilutes the embedding.
- **Query rewriting/expansion** — generate 2-3 alternate phrasings of an ambiguous query and retrieve for each, merging results. Useful for vague or underspecified questions; adds latency and cost, so reserve it for query types measured to need it.

## Reranking

A cross-encoder scores each `(query, candidate)` pair directly (more accurate, much slower) rather than comparing pre-computed embeddings (fast, less precise). Use it as a **second stage** over the top 20-50 candidates from initial retrieval — never as the first-pass retriever, it doesn't scale to a full corpus.

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")
pairs = [(query, c["text"]) for c in candidates]
scores = reranker.predict(pairs)
reranked = sorted(zip(candidates, scores), key=lambda x: x[1], reverse=True)
```

A larger `top_k` at the first stage is not a substitute for reranking or for fixing a retrieval failure — it adds noise to the evidence budget (see `context-assembly-and-citations.md`) without necessarily lifting the correct source higher.

## Deduplication

Multiple chunks from the same source document, or near-duplicate windows created by chunk overlap, can crowd out independent evidence in a fixed top-k. Deduplicate on document identity (and near-duplicate text similarity) before spending evidence-budget slots, not after.

## Metadata filtering

Pre-filter by metadata (department, document type, classification, effective date, tenant/company) before or alongside the vector/keyword query, not as a post-hoc check on the answer. Filtering *after* retrieval is a common security bug: the document was already visible to the retrieval layer and may leak through logs, ranking signals, or a "similar documents" feature even if the final answer text is scrubbed. See `production-reliability.md` for the access-boundary implications.

A request-supplied workspace, tenant ID, collection name, label, or prompt instruction is not authorization. Resolve the principal's allowed collection/document scope on the server, then apply it before every retrieval branch. Labels are useful routing metadata but are not proof that the caller may read a document.

## Diagnosing a retrieval failure

Work top-down, cheapest checks first:

1. **Is the expected source in the corpus/index at all?** (ingestion problem, not retrieval)
2. **Does it survive chunking with its identity intact?** (`chunking-strategies.md`)
3. **Does query handling need to extract/normalize something the raw query obscures?**
4. **Does one signal (vector or keyword) find it while the other misses?** → targeted fix for that signal, not an architecture change.
5. **Is it found but ranked outside top-k?** → reranking or metadata boosting, not a bigger index.
6. **Is it retrieved correctly but the answer still wrong?** → that's not a retrieval failure anymore, move to context assembly / generation.

## Anti-patterns

- **Fixed chunk size regardless of document type** — see `chunking-strategies.md`.
- **Embedding everything with one model regardless of content type** — validate per content type before assuming one embedding model fits code, prose, and tables equally.
- **Raising `top_k` to paper over a ranking problem** — it inflates the context budget and rarely fixes the root cause.
- **Same retrieval strategy for every query type** — reference lookups, cross-lingual questions, and multi-hop questions have different failure modes and need different first hypotheses (see the decision-hints pattern in `evaluation.md`).
- **No evaluation separating retrieval quality from answer quality** — see `evaluation.md`; without this split you cannot tell whether a bad answer is a retrieval bug or a generation bug.
