# RAG Engineer

`rag-engineer` is a portable skill for building, debugging, reviewing, and operating retrieval-augmented generation (RAG) systems. It helps an agent investigate the whole evidence pipeline instead of trying to solve every answer problem with prompt changes.

This package is guidance, not an application template. It does not assume a particular framework, vector database, embedding provider, company, or API shape.

## What is included

```text
SKILL.md                                      Core workflow and routing
references/multi-company-architecture.md     Tenant, identity, and lifecycle rules
references/chunking-strategies.md             Chunk design and indexing checks
references/embedding-models.md                Embedding and refresh decisions
references/retrieval-patterns.md              Retrieval, filtering, and reranking
references/context-assembly-and-citations.md  Evidence assembly and citations
references/evaluation.md                      Retrieval and answer evaluation
references/production-reliability.md          Security, latency, cost, and outages
```

## Installation

Clone this repository and place the skill directory where your coding agent discovers project skills.

For Claude Code:

```text
your-project/.claude/skills/rag-engineer/
```

For Codex:

```text
your-project/.codex/skills/rag-engineer/
```

The directory must contain `SKILL.md` at its root. Keep the `references/` directory beside it.

## First use in a real repository

1. Read `SKILL.md` and `references/multi-company-architecture.md`.
2. Inspect the actual application before choosing a database, chunk size, embedding model, or retrieval topology.
3. Create a repository-specific adapter at `references/<your-app>-implementation.md`.
4. Record verified entry points, retrieval backends, authorization filters, context assembly, citation logic, configuration, tests, and known failure modes in that adapter.
5. For each incident, locate the first failing stage in this order:

   ```text
   source → parse/chunk → embed/index → authorize/filter → retrieve
          → deduplicate/rerank → assemble → generate → cite → evaluate
   ```

6. Read only the reference file relevant to that stage, then prove the diagnosis with a deterministic test or metric.

Do not put company paths, endpoints, collection names, prompts, or vendor configuration into the portable reference files. Keep those details in the implementation adapter.

## Realistic example: an SOP assistant

Imagine a manufacturing company has PDF safety procedures in separate collections for `Plant A`, `Plant B`, and contractor documents. A technician asks:

> “What checks are required before restarting the packaging line after an emergency stop?”

A safe investigation and implementation looks like this:

1. **Ingest:** parse the PDF, preserve headings and page numbers, and split it into structure-aware chunks. Store stable `document_id`, `document_version_id`, and `chunk_id` values with the source URI and access-policy reference.
2. **Authorize:** authenticate the technician and resolve the collections they may access on the server. Do not treat a request-supplied plant name as authorization.
3. **Retrieve:** run vector search for the meaning of “restart after an emergency stop.” Add keyword search only if the measured question set shows that exact procedure IDs or acronyms are being missed.
4. **Inspect raw results:** confirm that the expected procedure appears before deduplication, reranking, or context trimming. If it is absent here, investigate parsing, chunking, embeddings, filters, or indexing.
5. **Assemble evidence:** pass the relevant chunks to the model with source identity, page/section metadata, and an explicit evidence budget.
6. **Answer and cite:** require the answer to stay within the retrieved evidence and cite the exact document version and page/section used. If the evidence is insufficient, refuse or ask for clarification.
7. **Evaluate:** add this question to a deterministic test set covering retrieval recall, citation fidelity, refusal behavior, and access-control negatives.

The important checkpoint is that every answer claim must be traceable to evidence the model actually received. A prompt rewrite cannot recover a procedure that was never retrieved, and a citation to an inaccessible document is a security failure even if the answer is factually correct.

## Common use cases

| Situation | Start with |
|---|---|
| Answers say “I don't know” | `chunking-strategies.md`, then `retrieval-patterns.md` |
| The right document exists but is not retrieved | `retrieval-patterns.md` and `embedding-models.md` |
| The answer is wrong despite relevant hits | `context-assembly-and-citations.md` |
| Users can see another company's documents | `multi-company-architecture.md` immediately; add raw-retrieval negative tests |
| Citations are missing or unverifiable | `context-assembly-and-citations.md` |
| Retrieval is slow or a backend is failing | `production-reliability.md` |
| Choosing hybrid search, reranking, or graph search | `retrieval-patterns.md`; measure before adding complexity |
| Changing chunking or embeddings | `chunking-strategies.md`, `embedding-models.md`, and `evaluation.md` |
| Preparing a pilot or release | `evaluation.md` and `production-reliability.md` |

## Useful prompts for an agent

Use concrete requests that identify the symptom and the repository:

```text
Use the rag-engineer skill. Trace why this question returns no evidence.
Start at the live entry point, inspect raw retrieval before context assembly,
and report the first failing pipeline stage with a deterministic test.
```

```text
Use the rag-engineer skill to review this RAG application for multi-company
authorization. Verify the server-side tenant boundary, raw retrieval filters,
canonical document identity, citation scope, and negative access tests.
```

```text
Use the rag-engineer skill to evaluate this proposed embedding change. Compare
the old and new versions on the same question set, including retrieval recall,
citation fidelity, refusal accuracy, latency, and cost.
```

## Definition of done

Before calling a new company RAG system ready, verify:

- retrieval works on representative question shapes;
- authorization filters are resolved server-side before retrieval;
- unauthorized documents are absent from raw results;
- canonical IDs survive every backend and citation step;
- deletion, re-indexing, and embedding-version changes are testable and idempotent;
- backend outages are distinguishable from healthy empty results;
- citations point to evidence shown to the model;
- evaluation covers accuracy, refusal, latency, cost, and degradation.

When the code changes, update `references/<your-app>-implementation.md` with newly verified file and line references. The portable core should remain reusable for the next company.

## License

MIT — see [LICENSE](LICENSE).
