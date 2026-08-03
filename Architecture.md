# Company Brain (CB) — Enterprise Retrieval Architecture

> A multi-tenant, permission-aware, retrieval-augmented system that acts as
> the **organizational memory** of a company. It indexes every document,
> email, ticket, wiki page, code repo, and chat the company has, and lets
> any authorized employee ask natural-language questions and get a cited
> answer in <2 seconds.

## Why this exists

LocalRAG1 solved the *personal* retrieval problem: one user, one folder,
one machine, keyword search good enough. It deliberately avoided
embeddings, vector stores, LLMs-on-every-query, and multi-user concerns.

**Company Brain solves a different problem:**

- **Heterogeneous sources** — Google Drive, Confluence, Jira, Slack, GitHub,
  SharePoint, Gmail, Salesforce, Notion, S3, local files. A retrieval
  system that only sees one folder is a toy.
- **Multi-tenant + permissions** — what a junior in Marketing is allowed
  to see is a strict subset of what the CFO sees. Retrieval must enforce
  the company's ACLs at *query time*, not at login time.
- **Natural-language answers, not just file lists** — when someone asks
  *"what's our policy on vendor data retention?"*, the right response is
  a paragraph synthesizing three policy docs, not 30 filenames.
- **Citations and provenance** — every answer must point to the source
  documents with chunk-level granularity. Hallucination is unacceptable
  in a system people will use to make business decisions.
- **Refresh cadence** — corporate knowledge changes daily. Indexing
  must be incremental, near-real-time, and survive source outages.
- **Auditable** — who asked what, what was retrieved, what was shown,
  what was answered. Required for regulated industries.

LocalRAG1 is the *kernel*; CB is the *operating system* built around it.

---

## Design principles

1. **Hybrid retrieval is mandatory** — neither BM25 nor vectors alone hit
   the recall bar on enterprise corpora. We use both, fused.
2. **Reranking is mandatory** — first-stage retrieval is recall-oriented;
   a cross-encoder reranker in the second stage handles the precision
   gap. This is the single biggest recall/precision lever.
3. **Permissions at the retrieval layer, not the UI** — every retrieval
   candidate carries an ACL; the reranker, the LLM, and the response
   formatter all see only authorized candidates.
4. **Citations as a first-class output** — every LLM answer is a list of
   `(chunk_id, doc_id, source_uri, span, score)` tuples. The UI shows
   them inline. The eval suite measures citation accuracy.
5. **The LLM is a *renderer*, not a *retriever*** — the LLM never decides
   what to find. It only decides how to phrase what's already retrieved.
   This is the core defense against hallucination.
6. **Cheap path, slow path, fallback** — most queries are answered by
   the cheap path (lexical + small vector lookup). Hard queries escalate
   to expensive retrieval + LLM synthesis. LLM is never the first move.
7. **Observable end-to-end** — every query emits a trace: which sources
   were queried, which candidates were returned, which were filtered by
   ACL, which were reranked, which were cited, what the LLM said, what
   the user clicked. No black boxes.
8. **Every subsystem with more than one realistic implementation is
   pluggable** — behind a thin interface, selected by configuration,
   addable without touching callers. Lexical index, vector index,
   graph index, reranker, LLM, every input source, embedder, object
   store, identity provider — all swappable via a config value.
   See [`DesignPrinciples.md`](./DesignPrinciples.md) for the
   pattern, registry structure, per-subsystem pluggability table,
   anti-patterns, and the four-question acceptance test. The rule
   of thumb: if a second implementation can be added only by editing
   the caller, the design is wrong — refactor to a Protocol first.

---

## System overview

```
                            ┌──────────────────────┐
                            │  Web UI / Slack /    │
                            │  Teams / API client  │
                            └──────────┬───────────┘
                                       │  (auth token, query, ctx)
                                       ▼
┌────────────────────────────────────────────────────────────────────┐
│                          QUERY SERVICE                             │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────────┐    │
│  │ Query normal- │   │ ACL resolver │   │ Query router         │   │
│  │ izer (Lang-  │   │ (per-user    │   │ (cheap / deep /      │   │
│  │  detect,      │   │  permission  │   │  agent)              │   │
│  │  intent)      │   │  scope)      │   │                      │   │
│  └──────┬───────┘   └──────┬───────┘   └──────────┬───────────┘   │
│         │                  │                      │               │
└─────────┼──────────────────┼──────────────────────┼───────────────┘
          │                  │                      │
          ▼                  ▼                      ▼
┌────────────────────────────────────────────────────────────────────┐
│                       RETRIEVAL PLANE                              │
│                                                                    │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────┐    │
│  │  Lexical index │  │ Vector index   │  │  Graph index       │    │
│  │  (OpenSearch / │  │ (Qdrant /      │  │  (Neo4j: people,   │    │
│  │   Elasticsearch│  │  Milvus /      │  │   docs, projects,  │    │
│  │   + BM25)      │  │  pgvector)     │  │   topics, edges)   │    │
│  └────────┬───────┘  └────────┬───────┘  └─────────┬──────────┘    │
│           └──────────────┬────┴────────────────────┘               │
│                          ▼                                         │
│                 ┌────────────────────┐                             │
│                 │  Hybrid fusion     │  RRF or weighted sum        │
│                 │  (BM25 + vector)   │                             │
│                 └─────────┬──────────┘                             │
│                           ▼                                        │
│                 ┌────────────────────┐                             │
│                 │ Cross-encoder      │  bge-reranker-v2 /          │
│                 │ reranker           │  cohere-rerank / colbert    │
│                 └─────────┬──────────┘                             │
│                           ▼                                        │
│                 ┌────────────────────┐                             │
│                 │ ACL post-filter    │  drop unauthorized          │
│                 └─────────┬──────────┘                             │
└───────────────────────────┼────────────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────────────┐
│                         LLM PLANE                                  │
│  ┌──────────────────┐   ┌────────────────┐   ┌─────────────────┐   │
│  │  Context builder │──►│  Answer LLM    │──►│ Citation        │   │
│  │  (top-K chunks,  │   │  (Claude /     │   │ extractor +     │   │
│  │   structured     │   │   GPT / Llama) │   │ verifier        │   │
│  │   prompt)        │   │                │   │                 │   │
│  └──────────────────┘   └────────────────┘   └─────────────────┘   │
│                                                                    │
│   Optional: agent loop for queries that need follow-up retrieval  │
└────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────────────┐
│                       INDEXING PLANE                               │
│  Connectors → Normalizer → Chunker → Embedder → Multi-index writer │
│  (Drive, Confluence, Jira, Slack, GitHub, Gmail, Salesforce, S3)    │
└────────────────────────────────────────────────────────────────────┘
```

---

## The four planes

A query flows top-down through four planes; an index update flows
bottom-up. Each plane is independently deployable and scalable.

### 1. Query plane — entry point

- **Query normalizer**: spell-correction, language detection
  (`lingua` or `fasttext`), intent classification (factual lookup /
  summarization / comparison / procedural), entity extraction
  (project names, people, dates — these become ACL + filter hints).
- **ACL resolver**: looks up the user's identity, group memberships,
  and document-level permissions from the identity provider (Okta /
  Azure AD). Produces a *permission scope* — a set of `(doc_id,
  allow | deny)` rules used as a post-filter on every candidate.
- **Query router**: classifies the query by complexity:
  - **Cheap path** — high-confidence lexical or single-source vector
    hit. No LLM. Returns top-K with snippets.
  - **Deep path** — hybrid retrieval + rerank + LLM synthesis with
    citations. Default for natural-language questions.
  - **Agent path** — multi-step retrieval (the LLM may call
    `retrieve(query, filter)` multiple times, like Tier 3 in LocalRAG1
    but with the full retrieval plane under it). Used for complex
    multi-hop questions (*"compare the data retention clauses in our
    top 5 vendor contracts"*).

### 2. Retrieval plane — finding candidates

Three indices, each optimized for a different retrieval signal:

#### Lexical index (OpenSearch / Elasticsearch + BM25)

- **Why BM25 alone is insufficient, but still necessary**:
  - Catches exact identifiers, error codes, project codenames, customer
    names, version numbers — things embeddings sometimes mangle.
  - 10–100× faster than vector search for keyword-style queries.
  - Cheap pre-filter before vector search: keyword hit narrows the
    vector search space.
- **Stack**: OpenSearch with the `BM25` similarity, custom analyzers
  per source type (code → identifier tokenizer; legal → preserve
  citation forms; chat → preserve emoji + usernames).
- **Per-document boost signals**: document recency, source authority
  (HR policies > random Slack messages), author reputation.

#### Vector index (Qdrant / Milvus / pgvector)

- **Why we need it**: semantic matches that BM25 misses entirely.
  *"vendor data retention"* matches a doc that says *"third-party
  records storage duration"*. *"how do I expense a client dinner"*
  matches a doc titled *"Reimbursable Entertainment Expenses"*.
- **Chunking strategy** (this is where most enterprise systems fail):
  - **Code** → AST-aware chunks (functions, classes), with file path
    and symbol as metadata.
  - **Markdown / Confluence** → heading-bounded chunks, with the
    heading hierarchy preserved.
  - **PDFs** → page-bounded, with a sliding window of 200 tokens and
    40-token overlap. Tables get special handling (extract as
    structured rows + a separate table-summary chunk).
  - **Chat (Slack / Teams / email threads)** → thread-bounded chunks,
    with participants and timestamp metadata.
  - **Spreadsheets** → per-sheet, per-region chunks (headers + a
    window of rows), NOT per-cell. The chunk text is a natural-
    language serialization of the region.
- **Embedding model**: pick by tenant requirements:
  - Default: `bge-large-en-v1.5` (1024 dim, strong general English)
  - Code-heavy tenants: `voyage-code-3` or `bge-code`
  - Multilingual: `bge-m3` (100+ languages, strong on mixed corpora)
  - Air-gapped / on-prem: `bge-small-en-v1.5` quantized
- **Index choice**:
  - **Qdrant** — recommended. Rust core, payload filtering is fast
    (filter by ACL during search, not after).
  - **Milvus** — for >100M vectors, distributed.
  - **pgvector** — fine for <10M vectors, when the company already
    has Postgres for everything else.
- **Per-vector metadata**: `doc_id`, `chunk_id`, `source_uri`,
  `acl_tags[]`, `mtime`, `chunk_text`, `embedding`. ACL filtering
  happens at the vector DB level via payload filter (Qdrant) or a
  pre-filter + post-filter pattern.

#### Graph index (Neo4j / Memgraph)

- **Why a graph at all**: some queries are inherently relational.
  *"Who owns the auth service?"*, *"what projects depend on the
  payments API?"*, *"what did team X work on last quarter?"*
- **Nodes**: People, Projects, Documents, Topics, Repositories, Tickets.
- **Edges**: `OWNS`, `MEMBERS_OF`, `DOCUMENTED_IN`, `DEPENDS_ON`,
  `REFERENCED_IN`, `MENTIONED_WITH`, `DISCUSSED_IN`.
- **Built from**: GitHub commit authorship, Confluence page ownership,
  Jira ticket assignment, Slack channel membership, document
  co-occurrence, link extraction. The graph is *derived*, not
  primary — documents and embeddings remain the source of truth.
- **Used for**: ACL inheritance, "people like you also asked",
  related-document expansion (follow edges to expand the candidate
  set before reranking), team-scoped search.

#### Hybrid fusion

BM25 and vector results are merged via **Reciprocal Rank Fusion (RRF)**:

```
score(d) = Σ  1 / (k + rank_i(d))     for each retriever i
```

- k = 60 (standard)
- Optional: weighted by retriever confidence (low-confidence BM25
  contributes less; this is learned from click data over time).
- Optional: per-source-type weights (legal docs weight vector more
  heavily; code weighs BM25 more).

#### Cross-encoder reranker

The fused top-200 candidates are reranked by a cross-encoder:

- **Model**: `bge-reranker-v2-m3` (multilingual, strong) or
  Cohere Rerank 3 (hosted, very strong but per-query cost).
- **Input**: `(query, chunk_text)` pair.
- **Output**: relevance score 0–1.
- **Cost**: ~50ms per 100 candidates. Cheap enough to do on every
  deep-path query.
- **Why this is the single biggest win**: BM25 + vector recall is
  usually 70–85% (high recall, noisy). After reranking, top-10
  precision is usually 90%+. The user sees the top-10.

#### ACL post-filter

The reranked top-K is filtered against the user's permission scope:

- Drop candidates where the user has no read access.
- If filtering empties the result, do *not* silently return nothing —
  surface "you don't have access to any matching documents" (this
  is important UX).
- ACL evaluation uses the **indexed ACL metadata**, not a live
  re-check against the source system (which would be too slow).
  A background job reconciles ACL changes.

### 3. LLM plane — answering

- **Context builder**: takes the top-K reranked chunks, deduplicates
  adjacent chunks from the same document, fits them into the LLM's
  context window with budget for prompt + answer.
- **Answer LLM**:
  - Default: Claude Sonnet or GPT-4o class
  - On-prem: Llama 3.1 70B or Qwen 2.5 72B
  - Streaming responses for UX
  - System prompt includes: identity ("you are Company Brain, you
    answer based on the provided context, you never invent"), the
    user role, the format requirement (markdown + citations), and
    the refusal policy.
- **Citation extractor + verifier**:
  - The LLM is prompted to cite by `[doc_id:chunk_id]` markers.
  - Post-generation, the citations are parsed and *verified*:
    - Every cited chunk must be in the context the LLM was given.
    - Every cited claim must have a matching text span in the cited
      chunk (we verify via substring match + a small entailment
      model).
  - Citations that don't verify are stripped and the answer is
    re-generated, or the claim is removed. **The user never sees an
    unverifiable citation.**
- **Agent path** (optional): for multi-hop queries, the LLM can call
  `retrieve(query, filters, recency)` multiple times within an
  iteration budget (default 6). Each call goes through the same
  retrieval plane. This is Tier 3 from LocalRAG1, but with the full
  retrieval stack underneath, not just FTS5.

### 4. Indexing plane — keeping the brain fresh

```
Source (Drive / Confluence / Slack / Jira / GitHub / ...)
    │
    ▼
Connector (per source, with auth + delta detection)
    │
    ▼
Normalizer (extract text + metadata → unified Doc schema)
    │
    ▼
Chunker (per-format chunking, see above)
    │
    ▼
Embedder (batched, GPU-pinned, async)
    │
    ▼
Multi-index writer (writes to BM25 index, vector index, graph, blob store)
    │
    ▼
ACL tagger (resolves per-chunk ACL from source metadata)
```

#### Connectors

- One per source system. Each implements:
  - **Auth**: OAuth2 / service account / app token, refreshed
    centrally via Vault.
  - **Initial sync**: full crawl with backpressure and rate limit
    handling.
  - **Delta sync**: per-source change feed (Drive Changes API,
    Confluence webhooks, Slack events API, GitHub webhooks, Jira
    webhook, IMAP IDLE for email).
  - **Resumability**: per-source cursors persisted in Postgres; a
    connector can be killed and resumed at any point.
  - **Error handling**: per-source retry policy, dead-letter queue
    for poison documents, circuit breaker for source outages.

#### ACL as a first-class concern

This is the part that distinguishes enterprise from personal. For
each chunk:

- Resolve the source document's ACL (from the source system, not
  inferred).
- Resolve *inherited* ACL (e.g., the folder the file is in).
- Resolve *negative* ACL (private docs, drafts, "eyes only").
- Resolve *group* ACL (the user is in `engineering-allhands`, the
  doc is shared with that group).
- Store the **denormalized ACL** as metadata on every chunk. We do
  *not* resolve at query time against the source system — too slow,
  too fragile.

The denormalized ACL is periodically reconciled against the source
of truth (background job, every N hours, configurable per source).

#### What we explicitly do not do at index time

- **Summarize**. The LLM is not in the indexing hot path. It's slow,
  expensive, and changes the text (which hurts BM25 recall and
  citation faithfulness). If a summary is useful, it's a *separate
  field* on the document, generated on a schedule, not inline.
- **Translate**. Same reason.
- **Generate embeddings for the LLM**. The LLM is the LLM; it
  reads the chunk text directly.

---

## Deployment model

### SaaS multi-tenant

- One logical deployment, multiple customers.
- Strict tenant isolation at the index level (per-tenant namespace
  in Qdrant, per-tenant index alias in OpenSearch).
- Tenant-level rate limits, per-user quotas, per-tenant model
  selection (some customers get hosted Cohere rerank, others get
  open-source rerank on our GPU pool).

### Self-hosted / on-prem

For regulated customers (finance, healthcare, government):

- The whole stack runs in the customer's VPC.
- The "LLM plane" runs against either:
  - Customer-provided Claude / OpenAI / Azure OpenAI endpoint
  - Customer-hosted Llama / Qwen on their GPU
- The retrieval plane is always customer-hosted (Qdrant +
  OpenSearch + Neo4j are all single-binary or simple-cluster
  deploys).
- We ship a Helm chart / Terraform module / Docker Compose stack.
- Telemetry goes *out* of the customer's environment only with
  explicit opt-in (and only aggregate metrics, never query content).

### Hybrid

- Indexing + retrieval on-prem.
- Answer LLM in the cloud (with chunk-level redaction before the
  chunk leaves the customer's perimeter — strip PII, apply customer
  redaction rules).

---

## Stack (recommended, opinionated)

| Component | Choice | Why |
|---|---|---|
| API / orchestration | Python (FastAPI) or Go | FastAPI for ML-ecosystem access; Go for raw throughput. Pick one. |
| Lexical index | OpenSearch | Mature, good BM25, good per-field analyzers, per-tenant aliases, easy to operate. |
| Vector index | Qdrant | Fast payload filtering (= ACL at search time), Rust core, simple ops, multi-tenant. |
| Graph index | Neo4j (or Memgraph) | For people/project/topic relations. Skip this for v1; add when ACL inheritance or "related" features become a requirement. |
| Embedding model | `bge-large-en-v1.5` (default) or `bge-m3` (multilingual) or `voyage-code-3` (code) | Open-weights, strong on BEIR/MTEB, runs on a single A10/A100. |
| Reranker | `bge-reranker-v2-m3` (open) or Cohere Rerank 3 (hosted) | Cross-encoder rerank is the biggest single quality lever. |
| Answer LLM | Claude Sonnet (cloud) or Llama 3.1 70B (on-prem) | Pick per deployment model. |
| Object store | S3 / GCS / Azure Blob | Stores originals for citation rendering (snippets, page images, highlighted spans). |
| Identity | Okta / Azure AD SSO | The identity provider is the ACL source of truth. |
| Orchestration | Kubernetes | Standard. Single-region first, multi-region later. |
| Observability | OpenTelemetry → Grafana / Datadog | Every query emits a trace. |
| Eval | Custom harness + BEIR + a private company-specific set | Continuous eval against a held-out query set, run nightly on new retrieval params. |

---

## Retrieval quality — the eval pipeline

This is where enterprise systems win or die. Without continuous
evaluation, every change to chunking / embedding / reranker is a
gamble.

- **Public benchmarks**: BEIR, MIRACL (multilingual), CodeSearchNet.
  Used to pick embedding model, not to claim product quality.
- **Private golden set**: 500–2000 hand-labeled queries with
  relevant-doc judgments. Built over time from real user queries
  (with consent). This is the source of truth.
- **Metrics**:
  - **Recall@10, Recall@100** — first-stage retrieval quality.
  - **nDCG@10, MRR** — reranked top-10 quality.
  - **Citation precision / recall** — of the citations the LLM
    produces, how many are correct, and of the truly relevant
    chunks, how many were cited.
  - **Answer faithfulness** — entailment model checks each claim
    in the answer against the cited chunk.
  - **End-to-end** — human raters on a sample of 100 queries/week.
- **Regression suite** runs on every change to the retrieval or LLM
  config. Catches silent regressions before they reach users.

---

## Latency budget

The user expects <2s. The budget:

| Stage | Target |
|---|---|
| Query normalization + ACL resolve | 50 ms |
| Hybrid retrieval (BM25 + vector, parallel) | 200 ms |
| RRF fusion | 5 ms |
| Cross-encoder rerank (top-200) | 250 ms |
| ACL post-filter | 20 ms |
| Context build + LLM first token | 800 ms (streaming) |
| Citation verification | 50 ms (parallel with streaming) |
| **Total to first token** | **~1.4 s** |
| Total to full streamed answer | 2–4 s |

Streaming makes the perceived latency much lower than the total.

---

## Security and privacy

- **Encryption at rest** for all indices and object stores.
- **Encryption in transit** everywhere (mTLS between services).
- **Per-tenant key isolation** (BYOK for self-hosted).
- **PII redaction** at the chunking stage (configurable patterns:
  SSN, credit card, employee IDs, etc.). Original redacted text is
  never sent to the LLM.
- **Query log retention** is per-tenant configurable; some
  regulated tenants have it set to 0 (no logs, full audit only
  via the identity provider).
- **Right-to-be-forgotten**: per-user / per-doc deletion is
  supported across all indices within a defined SLA (default
  24h, contractually negotiable).

---

## What CB explicitly is NOT

- ❌ A chatbot UI over an LLM. The retrieval is the product.
- ❌ A RAG-framework clone (LangChain / LlamaIndex wrappers). The
  architecture borrows patterns but is built for *this* workload,
  not for general agent orchestration.
- ❌ A replacement for source-of-truth systems (Confluence, Jira,
  GitHub). CB reads from them; it does not own the data.
- ❌ A "chat with your docs" personal tool. LocalRAG1 is that.
  CB is the multi-tenant, permissioned, audited, enterprise
  version.

---

## Build phases (proposed order)

1. **Phase 1 — Retrieval MVP (one source, one tenant)**
   - Single connector (Google Drive or Confluence)
   - BM25 + vector + reranker pipeline
   - Simple web UI (query box, ranked results, snippets)
   - No LLM synthesis yet — just retrieval quality work
   - Goal: prove that hybrid retrieval + reranking beats lexical
     alone on a real corporate corpus

2. **Phase 2 — LLM synthesis with citations**
   - Add the LLM plane
   - Citation verification pipeline
   - Streaming UI with inline citations
   - Eval harness with a private golden set
   - Goal: ship the answer-with-citations UX

3. **Phase 3 — Permissions and multi-tenancy**
   - ACL tagging at ingest
   - ACL post-filter at retrieval
   - Identity provider integration
   - Multi-tenant isolation
   - Goal: make this safe to use in a real company

4. **Phase 4 — Additional connectors**
   - Slack, Jira, GitHub, Gmail, Salesforce, Notion, S3
   - Connector framework + per-source ACL reconciliation
   - Goal: cover the 80% of corporate knowledge

5. **Phase 5 — Agent path for multi-hop queries**
   - LLM-driven iterative retrieval
   - Loop budgets, wall-clock caps, audit trail
   - Goal: handle the long tail of complex questions

6. **Phase 6 — Graph layer**
   - People / project / topic relations
   - Related-document expansion
   - "People like you also asked" features
   - Goal: support relational queries ("who owns X?")

7. **Phase 7 — Continuous eval and improvement**
   - Production telemetry → training data
   - Per-tenant retrieval quality dashboards
   - Automated regression on retrieval config changes
   - Goal: never ship a silent quality regression again

---

## The relationship to LocalRAG1

| | LocalRAG1 | Company Brain |
|---|---|---|
| Scale | 1 user, 1 folder, 1 machine | N users, M sources, 1 org |
| Retrieval | Lexical (FTS5 trigram) | Hybrid (BM25 + vector + rerank + graph) |
| LLM | Tier 3 fallback, opt-in | Default renderer, with citations |
| Permissions | None (single user) | First-class |
| Connectors | Filesystem | Drive, Confluence, Slack, Jira, GitHub, ... |
| UI | CLI | Web + Slack/Teams + API |
| Install | Single .exe, no model | Kubernetes (cloud) or single VM (on-prem) |
| Eval | Smoke tests | Continuous regression on a private golden set |
| Citation | n/a | Required, verified, inline |
| Cost model | Free, local | Per-query LLM cost; per-tenant metering |

LocalRAG1 is the kernel — useful in itself for personal use. CB is
the OS that runs across the company. The two are not competing
products. A CB user might also have LocalRAG1 indexing their
personal notes folder; that's a different scope, different
permission model, different latency target.
