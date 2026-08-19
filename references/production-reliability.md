# Production Reliability: Failure Isolation, Latency/Cost, Security

RAG systems have more moving dependencies than a typical CRUD service (vector DB, keyword index, graph store, embedding API, completion API), so "what happens when one of them is down or slow" is a design decision, not an afterthought.

## Table of contents
- [Failure isolation and graceful degradation](#failure-isolation-and-graceful-degradation)
- [Fail closed vs. fail open](#fail-closed-vs-fail-open)
- [Latency and cost](#latency-and-cost)
- [Observability without leaking sensitive content](#observability-without-leaking-sensitive-content)
- [Security and access boundaries](#security-and-access-boundaries)
- [Prompt/data injection](#promptdata-injection)
- [Pre-pilot checklist](#pre-pilot-checklist)

## Failure isolation and graceful degradation

**Failure isolation** — one dependency's failure (say, the graph/summary backend) should not corrupt or silently hide the failure of the whole request. **Graceful degradation** — when safe, return a clearly-labeled reduced result (e.g. "vector-only search, hybrid search unavailable") rather than silently pretending full capability still exists.

An empty result is not enough to describe degradation. Return or record per-backend status, timeout, latency, and error class so operators and callers can distinguish "no matching evidence" from "the backend was unavailable." Keep fail-open decisions explicit for each retrieval branch.

The unsafe version of this (returning a normal-looking answer after quietly losing required evidence) is worse than an explicit error, because the user has no way to know the answer's evidentiary basis just got weaker.

## Fail closed vs. fail open

Decide per-dependency, not globally:

- **Fail closed** (refuse rather than answer) when the missing dependency could mean the evidence or authorization backing the answer can no longer be trusted: e.g. the LLM is down (no generation is possible at all), or a permissions/filter service is unreachable (you cannot confirm the user is authorized to see what retrieval would return).
- **Fail open with a clear label** when the missing dependency only narrows coverage without compromising trust — e.g. keyword search is down but vector search still works and the answer says so.

A fallback must never fabricate an answer to mask an outage. "The system is temporarily unable to answer" is always safer than a guess presented with normal confidence.

## Latency and cost

Track and report **p50 and p95 latency separately**: a fast median can hide a painful slow tail that makes the system feel broken for a meaningful fraction of real users, especially once multiple backend calls (vector + keyword + graph + rerank + generation) are chained per request.

Cost drivers worth attributing individually rather than lumping into one number: embedding calls, graph/LLM-based indexing calls, answer generation calls, judge/evaluation calls, and total context size (tokens correlate with both cost and latency).

Observability should let you explain *why* a request was slow or expensive after the fact: structured traces/metrics per backend call, not just a single end-to-end timer.

## Observability without leaking sensitive content

Logs and traces are a common, underrated leak vector. Never log full document text, secrets, or raw sensitive content. Log identifiers, counts, timings, and truncated/redacted snippets only. "It's excluded from Git" does not mean it's excluded from a debug log at runtime; check both.

## Security and access boundaries

A factually correct, well-cited answer is still unacceptable if it exposes a document, secret, or scope the requesting user is not authorized to see. Key principles:

- **Least privilege** — give users, services, and tools only the access they need, not the maximum convenient default.
- **Metadata ≠ authorization.** A document having a `department` field does not mean access is enforced. Authorization requires a server-side filter applied *before* retrieval, tied to the requesting user's actual permissions, not a prompt instruction, and not a post-hoc check on the generated answer (by then the model has already seen the content).
- **Negative testing is the actual proof.** The only way to validate an access boundary is a test that proves a specific ineligible document is *absent from retrieval results* for a given identity, not merely that the final answer text happened not to mention it (a near-miss phrasing could still leak it).
- **Data boundary** — a technical (not just configuration) separation between demo/staging data and production data/credentials, so a demo environment can never accidentally touch production secrets or vice versa.

For multi-company systems, the authorization contract should be visible in the request path: authenticated principal → server-resolved collection/document scope → pre-retrieval filters. A request-supplied `workspace` or tenant label is routing input at most; it is not proof of access. Test that unauthorized documents are absent from raw retrieval results, not merely absent from the final answer.

## Prompt/data injection

Any content that gets pulled from an untrusted source (a document in the corpus, a user's own free-text) and placed into the model's context is a potential injection vector: text designed to make the model ignore its instructions, exfiltrate other context, or take an unintended action. Treat retrieved document content as data, never as instructions, when constructing the prompt, and validate/sanitize where documents can be uploaded by less-trusted parties (e.g. supplier-submitted content).

## Pre-pilot checklist

- [ ] Every backend dependency has an explicit fail-closed/fail-open decision, not an unhandled exception.
- [ ] p50 **and** p95 latency are both reported, not just an average.
- [ ] Logs/traces never contain full document text or secrets.
- [ ] At least one negative access test exists per role/boundary (proves absence from retrieval, not just from the visible answer).
- [ ] Demo/staging and production data/credentials are technically separated, not just separated by convention.
- [ ] Retrieved document content is treated as data in the prompt, never concatenated in a way that could be mistaken for an instruction.
- [ ] The API exposes degraded-backend status separately from a healthy empty result.
