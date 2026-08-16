# Chunking Strategies

How source documents get split before indexing. This decision is upstream of everything else in a RAG system — a fact that never survives chunking can never be retrieved, cited, or answered correctly, no matter how good the embedding model or the prompt is.

## Table of contents
- [The core tension](#the-core-tension)
- [Strategy 1: Fixed-size / recursive character splitting](#strategy-1-fixed-size--recursive-character-splitting)
- [Strategy 2: Semantic chunking](#strategy-2-semantic-chunking)
- [Strategy 3: Structure-aware chunking (Markdown/HTML/PDF)](#strategy-3-structure-aware-chunking-markdownhtmlpdf)
- [Strategy 4: Hierarchical / parent-child chunking](#strategy-4-hierarchical--parent-child-chunking)
- [Overlap: what it buys you and what it costs](#overlap-what-it-buys-you-and-what-it-costs)
- [Identity co-location](#identity-co-location)
- [Choosing chunk size](#choosing-chunk-size)
- [Decision table](#decision-table)

## The core tension

Every chunking decision trades off two failure modes:

- **Chunks too small** → a fact loses the surrounding context (heading, document ID, version, date, exception clause) needed to interpret or cite it correctly. Retrieval finds the fact but the answer can't be trusted or attributed.
- **Chunks too large** → the embedding blurs multiple topics into one vector, so semantic search gets less precise, and the evidence budget in the final prompt fills up faster with less-relevant text per chunk.

There is no universal "right" chunk size — it depends on document structure and question shape. Treat chunk size/overlap as a hypothesis to test against your own evaluation set (see `evaluation.md`), not a constant to copy from a blog post.

## Strategy 1: Fixed-size / recursive character splitting

Split by character or token count, falling back through separators (`\n\n` → `\n` → `. ` → `" "`) so it prefers to break at paragraph/sentence boundaries when possible.

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=["\n\n", "\n", ". ", " ", ""],
    length_function=len,
)
chunks = splitter.split_text(document_text)
```

**Use when:** prototyping, logs/transcripts, or any corpus without reliable structure to key off.
**Watch out for:** it still can (and will) split mid-table, mid-list-item, or right before a heading if the separators don't line up with the document's actual structure.

## Strategy 2: Semantic chunking

Detect topic shifts using embedding similarity between adjacent sentences/paragraphs instead of a fixed size, so a chunk boundary falls where the *meaning* changes.

```python
# Sketch: compare consecutive sentence embeddings, cut when similarity drops
# below a threshold. Libraries: llama-index SemanticSplitterNodeParser,
# langchain-experimental's SemanticChunker.
```

**Use when:** long-form prose (articles, reports, narrative documentation) where meaning boundaries don't line up with any fixed size.
**Cost:** variable chunk sizes complicate context-budget planning, and it's slower (one embedding call per candidate boundary).

## Strategy 3: Structure-aware chunking (Markdown/HTML/PDF)

Split on the document's own structural markers — headings, sections, tables — and **carry the heading path into each chunk** so a chunk is never orphaned from the identity that makes it interpretable.

```python
# Markdown: split on heading levels, prepend the heading trail to chunk text
# so "the 45-day deadline" chunk still says which procedure it belongs to.
chunk_text = f"{h1} > {h2}\n\n{paragraph_text}"
```

**Use when:** SOPs, technical docs, anything with a real table of contents. This is usually the highest-leverage strategy for compliance/QA use cases, because the thing that makes an answer *citable* (which procedure, which version) is exactly the structure this approach preserves.
**Tables:** never split a table mid-row; either keep a table as one chunk or use a table-aware parser that understands rows as units.

## Strategy 4: Hierarchical / parent-child chunking

Index at two granularities: a small "child" chunk for precise retrieval matching, and a larger "parent" chunk (section or document) that gets pulled into context once the child chunk scores well. This is often called sentence-window or auto-merging retrieval.

```python
# Retrieve on the child (tight embedding match), but expand to the parent
# before handing evidence to the model, so the model sees full context
# without the embedding having to represent an entire section at once.
```

**Use when:** retrieval precision matters but a single sentence out of context reads as ambiguous or unciteable.
**Cost:** two indexes (or a parent-pointer scheme) to keep in sync during re-ingestion.

## Overlap: what it buys you and what it costs

Overlap (characters repeated between adjacent chunks) protects a fact that happens to fall near a boundary — without it, a fact split across two chunks may not fully appear in either one.

- Too little overlap → boundary facts get lost.
- Too much overlap → near-duplicate chunks compete for the same top-k slots, silently shrinking the diversity of evidence a query actually retrieves. A duplicate chunk in the top-5 is a wasted retrieval slot.

A common starting point is 10–20% of chunk size, but validate this against your eval set rather than assuming it.

## Identity co-location

The single most useful diagnostic for a chunking bug: **does the chunk that contains the answer fact also contain (or carry via metadata) the document identity that makes the fact citable?**

If a fact ("the closure deadline is 45 days") is retrieved and answered correctly but the citation is wrong, weak, or missing — chunking is usually the root cause, not the prompt or the citation logic. Check this before touching the generation layer.

## Choosing chunk size

Don't pick a number in the abstract — run a controlled experiment:

1. Fix everything except chunk size/overlap.
2. Re-index with configuration A vs. configuration B.
3. Measure: does the fraction of expected-source chunks that co-locate the required fact and the document identity go up or down?
4. Only after the offline/proxy signal looks better, re-run a real retrieval evaluation (`evaluation.md`) to confirm the change actually improves retrieval, not just the proxy metric.

## Decision table

| Document type | Recommended primary strategy | Typical size | Typical overlap |
|---|---|---|---|
| SOPs / technical docs / anything with headings | Structure-aware, heading-carried | 300–800 chars per section chunk | 10–15% |
| Long-form prose / reports | Semantic chunking | Variable | N/A (boundary-based) |
| Logs / transcripts / chat | Fixed-size / recursive | 500–1000 chars | 10–20% |
| Legal / contracts | Hierarchical (clause → section) | Clause-level child, section-level parent | Low (structure already bounds it) |
| Tables / structured data | Whole-table or row-group chunks | N/A — never split mid-row | N/A |
