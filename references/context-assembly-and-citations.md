# Context Assembly, Evidence Budgets & Citation Design

The layer between "retrieval found the right thing" and "the model actually used it correctly" is the most under-engineered part of most RAG systems, and a common source of bugs that get misdiagnosed as prompting or model problems.

## Table of contents
- [Why this layer exists as its own failure mode](#why-this-layer-exists-as-its-own-failure-mode)
- [Evidence budgets](#evidence-budgets)
- [Source survival](#source-survival)
- [Citation design](#citation-design)
- [Citing only what the model saw](#citing-only-what-the-model-saw)
- [Multi-hop answers](#multi-hop-answers)
- [Checklist](#checklist)

## Why this layer exists as its own failure mode

Retrieval can succeed (the right document is in the candidate set) while the answer still fails, because something between retrieval and generation dropped, truncated, reordered, or duplicated the evidence before the model ever saw it. Always check this layer *before* blaming the prompt or the model when a correctly-retrieved source produces a wrong or uncited answer.

## Evidence budgets

The context window is a shared, finite resource across every retrieved channel (vector hits, keyword hits, graph/summary hits, etc.). An "evidence budget" is the explicit policy for how that resource gets allocated:

- **Per-channel allocation** — e.g. reserve a minimum share for each backend so one noisy channel can't crowd out another's good hit.
- **Truncation strategy** — truncate whole low-relevance items before truncating mid-item; cutting a chunk in half is often worse than dropping it entirely, because a half-fact can be more misleading than an absent one.
- **Ordering** — some models attend unevenly across a long context (a "lost in the middle" effect); placing the highest-relevance evidence near the start or end of the context can measurably change answer quality on long-context prompts.

```python
def assemble_context(ranked_items, max_chars=8000):
    context_parts, total = [], 0
    for item in ranked_items:  # already sorted by relevance
        if total + len(item.text) > max_chars:
            break  # drop whole item rather than truncate mid-fact
        context_parts.append(f"[Source: {item.source_id}]\n{item.text}")
        total += len(item.text)
    return "\n\n---\n\n".join(context_parts)
```

## Source survival

**Source survival** = whether the evidence needed for a specific answer is still present in the *final assembled* context after formatting, deduplication, and truncation, as opposed to merely being present in the raw retrieval results. Track this as its own diagnostic:

> "All expected sources were retrieved, but one was truncated before generation" is a context-assembly bug, not a retrieval bug and not a generation bug. Fixing the prompt will not fix it.

## Citation design

A citation is only as trustworthy as three properties:

- **Citation identity** — a stable, reproducible reference (document + chunk/page) the user can actually go look up. Not just a filename if the corpus has revisions/versions.
- **Citation fidelity** — the cited source contains or supports the claim being made. A source can be *related* to the topic without actually supporting the specific claim. That's a subtler and more dangerous failure than an outright missing citation, because it looks correct.
- **Citation completeness** — every material claim in a multi-part answer has a supporting citation, especially for multi-hop answers that combine facts from more than one document.

## Citing only what the model saw

The most common citation bug: deriving the "sources" list from the *raw retrieval results* instead of from the *items that actually survived into the final prompt*. If context assembly dropped an item, it must not appear in the citation list. Otherwise the system claims to have used evidence it never showed the model.

```python
def sources_from_context_items(context_items):
    """Derive citations ONLY from what survived context assembly —
    never from the full raw retrieval set."""
    return [{"source": item.source_id, "chunk": item.chunk_id} for item in context_items]
```

## Multi-hop answers

When an answer requires combining facts from two or more documents, verify each material claim maps to *a* source, not just that *some* source appears somewhere in the citation list. A two-source question answered with only one citation is a red flag worth checking explicitly, not something to wave away because "a citation is present."

## Checklist

- [ ] Is there an explicit, testable evidence-budget policy (not just "whatever fits")?
- [ ] Does truncation drop whole items before truncating mid-item?
- [ ] Is there a "source survival" check separate from "was it retrieved"?
- [ ] Are citations derived from post-assembly context items, never raw retrieval hits?
- [ ] For multi-hop questions, is each claim checked against its own supporting source?
- [ ] Is citation identity stable enough for a user to actually verify (not just a bare filename)?
