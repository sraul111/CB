# Company Brain (CB) — Project Status

> Start here when resuming work on this project.
> For *why* the project exists and the design decisions, see
> [`Architecture.md`](./Architecture.md). This file focuses on
> **where we are** and **what to do next**.

---

## TL;DR

- **Architecture-only** as of this commit. No code yet.
- Target stack: FastAPI (or Go) for the API, OpenSearch (BM25),
  Qdrant (vectors), Neo4j (graph, Phase 6), `bge-large` (embeddings),
  `bge-reranker-v2` (rerank), Claude Sonnet / Llama 3.1 (answer LLM).
- The architecture is split into four planes: **Query, Retrieval,
  LLM, Indexing** — each independently deployable.
- First target: prove hybrid retrieval + rerank beats lexical on a
  real corporate corpus (Phase 1).
- See [`Architecture.md`](./Architecture.md) § "Build phases" for
  the 7-phase plan.

---

## What's done

| Phase | Item | Where |
|---|---|---|
| — | Architecture document | `Architecture.md` |
| — | Project README | `README.md` |
| — | This status file | `STATUS.md` |
| — | Discussion 1 — single-PC test plan + synthetic-data strategy | `Discussion1.md` |
| — | Standing working agreement (commit-don't-push, crisp answers) | `SystemInstruction.md` |
| — | Repo created and pushed on GitHub | `sraul111/CB` (private) |

Nothing else yet. No code, no infra, no eval harness.

## End-of-day handoff (this session)

- Architecture validated against the standard RAG course diagram — no
  changes needed; the course diagram is a subset of the design.
- `Discussion1.md` written: how to test the architecture on a single
  PC, including the synthetic-data plan (M365 Developer Sandbox +
  seed-and-multiply Python script for Jira/Slack/Confluence).
- `SystemInstruction.md` written: standing rules — I commit, user
  pushes; crisp answer style; session-start checklist.
- Repo pushed to `https://github.com/sraul111/CB` manually by the
  user after the initial commit landed locally.

**First thing next session:** continue architecture discussion. The
user wants more design talk before any code is written. Open
`SystemInstruction.md` and `Discussion1.md` to recover context.


## What's next (the actual first step)

**Phase 1 — Retrieval MVP (one source, one tenant).** Specifically:

1. **Pick a single source** to integrate first. Recommended: Confluence
   or Google Drive (highest-information content in most companies,
   well-documented APIs).
2. **Build a minimal indexing pipeline**:
   - Connector with OAuth2 auth
   - Text extraction (HTML for Confluence, native for Drive)
   - Chunking (heading-bounded for Confluence, page-bounded for Drive)
   - Embedding with `bge-large-en-v1.5` (or `bge-m3` if multilingual)
   - Write to: a local OpenSearch (Docker), a local Qdrant (Docker)
3. **Build a minimal query pipeline**:
   - BM25 query against OpenSearch
   - Vector query against Qdrant
   - RRF fusion
   - Cross-encoder rerank with `bge-reranker-v2-m3`
   - Return top-K with snippets
4. **Build a minimal web UI**: query box, ranked results, snippet
   highlights. No LLM yet — just retrieval.
5. **Build an eval harness**:
   - Hand-craft 50–100 queries with known-relevant docs
   - Measure Recall@10, nDCG@10, MRR
   - Compare: BM25 alone, vector alone, hybrid, hybrid+rerank
   - Goal: quantify the contribution of each stage

Only after Phase 1 hits a quality bar on the eval harness do we move
to Phase 2 (LLM synthesis + citations).

## Open architectural questions to resolve during Phase 1

1. **Python or Go for the API?** FastAPI is the path of least
   resistance (ML ecosystem, embedding/reranker clients all
   Python-first). Go wins on throughput. Recommend Python for the
   first 3 phases; revisit if the API tier becomes the bottleneck.
2. **Single-tenant or multi-tenant from day 1?** Recommend
   single-tenant for Phase 1 (one internal company) and refactor
   for multi-tenancy in Phase 3. Premature multi-tenancy is a
   common source of bugs in retrieval systems.
3. **Cloud or on-prem first?** Cloud is faster to ship. On-prem is
   the harder but more lucrative deployment. Recommend cloud for
   Phase 1–3, on-prem in Phase 4.
4. **Chunking strategy** — the spec in `Architecture.md` is
   per-format. Validate it on real data early; bad chunking is the
   #1 cause of poor retrieval and the hardest thing to refactor
   later.

## File map (what's where, planned)

```
CB/
├── Architecture.md          # read first
├── README.md                # user-facing intro
├── STATUS.md                # this file — read second
├── docs/                    # deeper write-ups as they're written
│   ├── chunking.md
│   ├── acl-model.md
│   ├── eval.md
│   └── deployment.md
├── src/                     # code, when written
│   ├── api/                 # FastAPI / Go service
│   ├── query/               # query plane
│   ├── retrieval/           # retrieval plane (BM25, vector, rerank, fusion)
│   ├── llm/                 # LLM plane (context builder, answer, citations)
│   ├── indexing/            # connectors, normalizer, chunker, embedder
│   ├── acl/                 # ACL resolver, tagger, reconciler
│   ├── eval/                # golden set, metrics, regression harness
│   └── lib/                 # shared types
├── connectors/              # one per source system
│   ├── confluence/
│   ├── drive/
│   ├── slack/
│   ├── jira/
│   ├── github/
│   ├── gmail/
│   └── salesforce/
├── deploy/                  # Helm chart, Terraform, Docker Compose
│   ├── helm/
│   ├── terraform/
│   └── docker-compose.yml
├── eval/                    # golden query sets + harness configs
│   ├── golden/
│   └── configs/
└── tests/                   # unit + integration + e2e
```

## Conventions (to be set in Phase 1)

- Pick a language (Python recommended) and stick to it for the
  first 3 phases.
- Every connector implements the same `Source` interface: `crawl()`,
  `delta()`, `resolve_acl(doc)`.
- Every retrieval result is the same `Candidate` shape: `(chunk_id,
  doc_id, source_uri, chunk_text, score, score_breakdown, acl_tags)`.
- Eval regressions fail CI.

## How to resume after a break

```bash
cd C:/Repos/CB
git log --oneline -10
cat STATUS.md                  # ← you are here
cat Architecture.md            # ← read this if you haven't recently
```

Then pick the next item from "What's next" above.
