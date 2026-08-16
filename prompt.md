Create an Excalidraw diagram explaining how the `rag-engineer` skill works.

Repository:
https://github.com/scheffershen/rag-engineer-skill

First inspect the repository, especially:
- `SKILL.md`
- `README.md`
- `references/multi-company-architecture.md`
- `references/chunking-strategies.md`
- `references/embedding-models.md`
- `references/retrieval-patterns.md`
- `references/context-assembly-and-citations.md`
- `references/evaluation.md`
- `references/production-reliability.md`

The diagram should explain the skill as a portable engineering workflow for building, debugging, reviewing, and
operating RAG systems—not as a standalone application.

Show the main flow from a user or engineering symptom to a verified diagnosis:

User symptom or task
→ inspect the real repository and live entry points
→ identify the first failing RAG pipeline stage
→ read only the relevant reference
→ prove the diagnosis with a deterministic test or metric
→ implement the smallest measured change
→ evaluate quality, safety, latency, cost, and reliability

Represent the core RAG evidence pipeline as the central horizontal flow:

source
→ parse / chunk
→ embed / index
→ authorize / filter
→ retrieve
→ deduplicate / rerank
→ assemble evidence
→ generate
→ cite
→ evaluate

Make each stage visually distinct and label the main failure question at each important checkpoint:

- Source: Was the source found and parsed?
- Parse/chunk: Was the required fact preserved in a useful chunk?
- Embed/index: Was the content embedded and indexed?
- Authorize/filter: Is the user allowed to access it?
- Retrieve: Did the expected source appear in raw results?
- Deduplicate/rerank: Did the source survive ranking and deduplication?
- Assemble evidence: Did it fit within the context budget?
- Generate: Did the model use the evidence faithfully?
- Cite: Does the citation point to evidence actually shown to the model?
- Evaluate: Which metric proves whether the change worked?

Add a prominent warning near the pipeline:

“Do not use a prompt change to fix a retrieval, authorization, citation, indexing, or backend-outage problem.”

Show the skill’s minimum safe architecture as a supporting group containing:

- parser
- structure-aware chunks
- canonical document and chunk IDs
- vector retrieval
- server-side access filters
- explicit context budget
- grounded generation
- stable citations
- deterministic evaluation

Show optional capabilities branching from the core architecture, but make clear that they are added only when
evaluation demonstrates a measurable need:

- keyword search for IDs and exact terms
- reranking when recall is good but ordering is poor
- graph search for measured multi-hop needs
- summary or parent retrieval for long documents
- vision indexes
- query rewriting

Add a separate security and multi-tenant authorization lane:

authenticated principal
→ server-resolved allowed collections/documents
→ pre-retrieval filters
→ retrieval
→ context assembly
→ generation

Clearly mark these as unsafe:

- trusting a request-supplied tenant or workspace
- retrieving broadly and filtering afterward
- relying on prompt instructions for authorization
- citing a document the model was not shown
- returning a normal-looking answer after silently losing a backend

Add an ingestion and lifecycle state machine:

discovered
→ parsed
→ chunked
→ indexed per backend
→ verified
→ active

Also show the failure/retry branch:

parsed or indexed
↘ failed / retryable

Include a feedback loop from evaluation back to the relevant pipeline stage. The evaluation section should show
separate metrics rather than one generic accuracy score:

- retrieval recall / union recall
- strict answer accuracy
- source hit rate
- citation fidelity
- refusal accuracy
- hallucination rate
- p50 and p95 latency
- per-request cost
- authorization negative tests
- re-indexing and deletion consistency

Show that unanswerable, partial-evidence, conflicting-evidence, and ambiguous-evidence questions are deliberate
safety tests.

Use visual semantics:
- blue for data and retrieval flow
- purple for model and generation steps
- red for security boundaries and failure risks
- green for verification, evaluation, and successful completion
- amber for optional components and measured complexity
- diamonds for decisions
- rectangles for processing stages
- thick boundary boxes for security zones
- dashed arrows for optional or conditional paths
- red crossed-out annotations for unsafe practices

Use a clean, modern hand-drawn Excalidraw style. The diagram should argue for the skill’s central principle:

“Retrieval quality bounds generation quality. A better prompt cannot recover a fact that was never indexed,
retrieved, authorized, or preserved in the final context.”

Organize the composition into three readable regions:

1. Left: symptoms, repository inspection, and routing to the correct reference
2. Center: the evidence pipeline and failure diagnosis checkpoints
3. Right: deterministic evaluation, production readiness, and feedback into the pipeline

Add a small side panel titled “Portable skill, repository-specific adapter” explaining:

- `SKILL.md` contains company-neutral principles and routing
- reference files contain reusable engineering guidance
- `references/-implementation.md` maps those principles to actual entry points, backends, filters, citation
logic, configuration, tests, and known failure modes
- company-specific paths, endpoints, collection names, prompts, and vendor settings should stay in the adapter

Keep the diagram focused enough to understand in one glance. Do not turn every reference file into an equally
prominent box. Prefer one central argument with grouped supporting details.

Output:
1. A `.excalidraw` JSON file.
2. A rendered PNG preview.
3. Visually validate the layout and fix overlapping arrows, unreadable labels, and excessive text before
finishing.