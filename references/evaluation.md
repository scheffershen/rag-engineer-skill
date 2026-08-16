# RAG Evaluation & Safety Testing

A RAG system is nondeterministic end-to-end, which means "it sounds right" is not a quality signal. This file covers how to build ground truth you can trust, and how to measure quality without fooling yourself.

## Table of contents
- [The evaluation tuple](#the-evaluation-tuple)
- [Separate retrieval quality from answer quality — always](#separate-retrieval-quality-from-answer-quality--always)
- [Metrics to read separately](#metrics-to-read-separately)
- [Metadata-only retrieval matching](#metadata-only-retrieval-matching)
- [Refusal, uncertainty, and hallucination](#refusal-uncertainty-and-hallucination)
- [Question-shape coverage](#question-shape-coverage)
- [Baseline hygiene](#baseline-hygiene)
- [RAGAS and automated judges](#ragas-and-automated-judges)

## The evaluation tuple

Every evaluation question is a contract, not just a user-style question:

```
question + category + expected_source(s) + required_fact(s) + expected_behavior
```

- **expected_behavior** for an answerable question = a correct, grounded, cited answer.
- **expected_behavior** for a deliberately unanswerable question = a refusal.

Ground-truth rules worth enforcing before trusting any score:
- Every expected source must actually exist in the corpus.
- Every required fact must literally occur in an expected source (not something the evaluator merely believes is true).
- Unanswerable questions must have no expected source and no required fact.
- Keep the evaluation set stable while comparing system changes — a moving target makes before/after comparisons meaningless.

## Separate retrieval quality from answer quality — always

A fluent, confident answer is not evidence the system works. The first diagnostic question for *any* wrong answer is: **was the evidence missing, or was it present but misused?**

| Observation | Likely layer | First place to inspect |
|---|---|---|
| Expected document absent from raw search results | Retrieval | Index, chunking, query form, thresholds, ranking |
| Expected document retrieved, but not cited | Context/citation assembly | Truncation, source extraction (`context-assembly-and-citations.md`) |
| Document cited, but answer omits/contradicts a required fact | Generation | Prompt, context organization, model behavior |
| Unanswerable question gets a confident answer | Grounding/safety | Refusal instruction, evidence threshold, eval set |
| Correct answer but slow | Operations | Backend latency, call count, context size (`production-reliability.md`) |

Never fix a generation-layer failure and a retrieval-layer failure with the same generic prompt tweak — they need different fixes, and conflating them wastes iteration cycles chasing the wrong layer.

## Metrics to read separately

Report these independently — never collapse them into a single "% accurate":

- **Retrieval union / recall** — did the combined indexes return every expected source, before any judge is involved?
- **Strict answer accuracy** — judge verdict is unambiguously correct.
- **Lenient accuracy** (`correct + partial`) — useful for diagnosis, not a substitute for strict accuracy in a headline claim.
- **Source hit rate** — every expected source actually cited in the final answer.
- **Refusal accuracy** — unanswerable questions correctly declined.
- **Hallucination rate** — unanswerable questions answered anyway (this and refusal accuracy are not the same axis — track both).
- **Latency p50 / p95** — see `production-reliability.md`.

A "good baseline statement" names the corpus, eval set version, model, configuration, and metric definition: *"On the N-question eval set, with model X and top-k Y, strict answer accuracy was Z%."* A bare "the chatbot is 90% accurate" is not reproducible and should not be trusted or repeated as a claim.

## Metadata-only retrieval matching

A subtle but important scoring bug: crediting a retrieval hit because the expected document ID appears somewhere in a *different* chunk's body text (a legitimate cross-reference), rather than in the retrieved item's own source metadata. This inflates retrieval scores artificially. Always match expected sources against structured metadata (`source`, `document_id`, `filename`), never against arbitrary chunk body text.

## Refusal, uncertainty, and hallucination

Unanswerable questions in an eval set are safety tests, not "bad questions" — they measure whether the system recognizes the boundary of what it actually knows.

- **Refusal** — a clear statement that the indexed evidence does not establish the requested fact.
- **Uncertainty** — a calibrated statement of what the evidence *does* support and what remains unknown (partial-evidence case).
- **Hallucination** — any substantive claim made without adequate support from the evidence actually shown to the model. A fluent *partial* answer can still hallucinate if it adds one unsupported detail — don't only test the all-or-nothing case.

Test all four shapes explicitly: no evidence, partial evidence, conflicting evidence, and ambiguous evidence.

## Question-shape coverage

A single "accuracy" number over a homogeneous question set hides which capability is actually weak. A good eval set covers distinct shapes and reports them separately where useful:

| Shape | Tests |
|---|---|
| Factual lookup | Precise number/name/date/requirement |
| Reference lookup | Exact identifier/SOP number retrieval |
| Multi-hop | Combining evidence across >1 document |
| Cross-lingual | Query and source in different languages |
| Acronym/jargon | Domain vocabulary resolution |
| Unanswerable | Refusal instead of rewarded hallucination |

A system can perform well on ordinary semantic questions while failing exact-identifier lookups, multi-document reasoning, or refusal — don't let a strong average mask a specific, fixable weak spot.

## Baseline hygiene

For every failure worth acting on, record: expected source, actual source(s) retrieved, and the judge's stated reason. This turns a percentage into an engineering hypothesis you can test a fix against, rather than a mood.

Test the *evaluator itself* before trusting its output — an evaluation harness can print a confident percentage while being wrong. Cases worth a scorer self-test: accented/non-ASCII text, multi-source questions, body-text cross-references (see above), missing citations, and refusals.

## RAGAS and automated judges

Frameworks like [RAGAS](https://arxiv.org/abs/2309.15217) formalize LLM-judged metrics (faithfulness, answer relevance, context precision/recall) on top of the same underlying idea: evaluate retrieval and generation as separate, measurable axes rather than judging only the final prose. Use an LLM judge to scale evaluation, but keep at least a small human-reviewed sample to catch judge drift or miscalibration, especially after a model or prompt change.

For a multi-company deployment, add deterministic gates that an LLM judge cannot replace:

- unauthorized documents are absent from raw retrieval results for each role/tenant boundary;
- expected document and chunk identities survive retrieval, deduplication, and context assembly;
- re-indexing, deletion, and embedding-model changes preserve canonical identity and do not leave orphaned derived records;
- failed retrieval backends are distinguishable from healthy empty results;
- source citations resolve to the exact evidence shown to the model.
