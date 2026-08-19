# Embedding Models & Vector Search

## Table of contents
- [What an embedding model actually buys you](#what-an-embedding-model-actually-buys-you)
- [Model selection guide](#model-selection-guide)
- [Vector database selection](#vector-database-selection)
- [Qdrant quick reference](#qdrant-quick-reference)
- [Cross-lingual and domain-specific caveats](#cross-lingual-and-domain-specific-caveats)
- [Embedding refresh](#embedding-refresh)

## What an embedding model actually buys you

Vector/semantic search finds chunks whose *meaning* is close to the query's meaning, even when the wording differs: paraphrases, synonyms, cross-lingual questions. It is weak at literal identifiers: SOP numbers, acronyms, exact names, dates, version strings. Full-text/keyword search is the mirror image. Neither alone is a complete retrieval strategy; see `retrieval-patterns.md` for combining them.

## Model selection guide

| Model | Dimensions | Notes |
|---|---|---|
| OpenAI `text-embedding-3-small` | 1536 | Good default: fast, cheap, strong general quality |
| OpenAI `text-embedding-3-large` | 3072 | Higher quality, higher cost/latency — use when retrieval precision is the bottleneck, not before |
| Cohere `embed-v4` / `embed-english-v3.0` | 1024 | Strong semantic-search-focused alternative, good multilingual variants |
| `all-mpnet-base-v2` (Sentence Transformers) | 768 | Self-hosted, no per-call cost, solid quality |
| `all-MiniLM-L6-v2` | 384 | Self-hosted, very fast, lower quality — fine for prototyping |
| Domain-specific (e.g. `codebert`, `biobert`, legal-BERT variants) | varies | Only worth the switch if you've *measured* general models underperforming on your domain's vocabulary |

Don't default to the largest/most expensive model. Measure retrieval quality on your own eval set first. A smaller model with good chunking often beats a larger model with bad chunking.

## Vector database selection

| Database | Best for | Notes |
|---|---|---|
| **Qdrant** | Self-hosted production | Rust core, strong filtering, named vectors (multiple embeddings per point), good for hybrid setups |
| **Pinecone** | Managed / rapid prototyping | No infra to run, pay-per-use |
| **Chroma** | Local development | In-process, zero infra, good for early prototyping only |
| **Weaviate** | Complex schemas, GraphQL | Strong when you need rich cross-object relationships |
| **Milvus** | Very large scale, distributed | Highest operational complexity — only reach for it once single-node Qdrant is genuinely insufficient |

## Qdrant quick reference

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct, Filter, FieldCondition, MatchValue

client = QdrantClient(url="http://localhost:6333")

client.create_collection(
    collection_name="documents",
    vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
)

client.upsert(
    collection_name="documents",
    points=[
        PointStruct(
            id=1,
            vector=[...],  # 1536 floats
            payload={
                "text": "chunk text",
                "source": "PR-QA-MRD-009",   # document identity metadata — see chunking-strategies.md
                "document_name": "...",
                "company": "...", "department": "...", "doc_type": "...",  # access/filter metadata
            },
        )
    ],
)

results = client.search(
    collection_name="documents",
    query_vector=[...],
    limit=5,
    score_threshold=0.7,
    query_filter=Filter(must=[FieldCondition(key="department", match=MatchValue(value="QA"))]),
)
```

**Score thresholds:** a raw similarity score is not a universal quality signal. It depends on the embedding model and the corpus. Calibrate a threshold against your own eval set rather than copying `0.7` from documentation.

## Cross-lingual and domain-specific caveats

- Cross-language retrieval quality depends entirely on whether the embedding model was trained multilingually. If a corpus mixes languages (e.g. French SOPs, English questions), verify this explicitly with a cross-lingual test case. Don't assume it works.
- Domain vocabulary (medical, legal, internal jargon/acronyms) can be poorly represented by general-purpose embeddings. If acronym or jargon questions underperform, that's evidence for either a domain-specific model or (often cheaper) a keyword/full-text signal layered on top; see `retrieval-patterns.md`.

## Embedding refresh

Embeddings go stale in two ways people forget to check for:
1. **Source document changed**, but the index wasn't re-run. The vector still represents the old text.
2. **Embedding model changed** — old vectors and new vectors are not comparable; a partial re-index mixing model versions in one collection silently corrupts similarity comparisons. Re-embed the whole collection, don't mix.
