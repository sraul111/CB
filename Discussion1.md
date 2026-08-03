# Discussion 1 — Testing Company Brain on a single PC

> A walkthrough of how to validate the Company Brain architecture
> end-to-end on one Windows machine, against a real personal corpus
> (local drives + Git repos + later, Google Drive).

## Context

The goal of this discussion: figure out how to *test* Company Brain on
the hardware and accounts already available, before any of the
enterprise concerns (multi-tenancy, on-prem deployment, full ACL)
become relevant.

**What we have:**

- One Windows PC, multiple local drives (C:, D:, ...)
- Local Git repositories
- A Google account (for a future Drive connector)

**What we do NOT need for this test:**

- Kubernetes (Docker Compose is enough)
- Neo4j / graph layer (Phase 6, no value at single-user scale)
- Multi-tenancy (one user = one tenant)
- Real ACL enforcement (treat the corpus as world-readable; bolt ACL
  on in Phase 3 when there is a second user to test against)
- Hosted rerankers or LLMs (local models are free and offline)

---

## The stack on one machine

| Component | Concrete choice | Why this one |
|---|---|---|
| **Language** | Python 3.11+, FastAPI | Already decided |
| **Vector DB** | Qdrant (single binary, Docker) | Fast payload filter = ACL at search time, as designed |
| **Lexical index** | OpenSearch (Docker) OR **SQLite FTS5** for v0 | FTS5 is "good enough" on a single machine and zero infra. Use OpenSearch the day BM25 across >100k docs is needed. |
| **Embeddings** | `bge-large-en-v1.5` via `sentence-transformers` on CPU (or GPU if available) | No external API, no cost, runs offline |
| **Reranker** | `bge-reranker-v2-m3` via `sentence-transformers` | Same — local, free |
| **LLM (answer)** | **Ollama** running `llama3.1:8b` or `qwen2.5:14b` | Local, no API key, no cost, no data leaving the PC |
| **Object store** | Local folder `data/blobs/` | Holds originals for citation snippets |
| **Identity / ACL** | Skipped (single user) | No analog in single-user mode; defer to Phase 3 |
| **Orchestration** | Docker Compose (one file) | Whole stack comes up with `docker compose up` |
| **Eval** | `eval/` folder with ~50 hand-written queries + a small script | The single most important thing to build early |

---

## Connectors in scope

| Source | Connector approach |
|---|---|
| **Local drives C:, D:, ...** | Filesystem connector that walks paths, skips junk (`node_modules`, `.git/objects`, `AppData`, ...), respects `.ragignore` (same syntax as `.gitignore`) |
| **Git repos** | Same filesystem walk, but on a cloned repo. Free bonus signal: file's git history (last-modified commit, author, blame) as metadata |
| **Google Drive** (later) | Drive connector reading "My Drive" via OAuth. Falls into the same `Normalize → Chunk → Embed → Write` pipeline. |

**Not worth connecting on day 1:**

- Gmail (rate limits + auth pain)
- Confluence / Jira / Slack (no instance available)

Add them when the corresponding account is in hand.

---

## What "testing it" actually means

Three testable deliverables, in this order. **Do not build all three
before running the first one.**

### 1. The retrieval MVP (Phase 1 on one machine)

**Goal:** prove that hybrid retrieval + rerank beats lexical alone on
the *real personal corpus*. This is the only experiment that
justifies the architecture.

- **Ingest:** point the filesystem connector at
  `C:\Users\SujitKumarRaul\Documents`, `C:\repos`, and a few project
  folders on `D:`. Skip binaries, skip `node_modules`, skip
  `AppData`.
- **Index:** chunk → embed → write to FTS5 + Qdrant (both, side by
  side).
- **Query:** a `curl` / `python -m cb query "what's the policy on X?"`
  that returns top-K with snippets.
- **Eval:** write 50 queries by hand, run them against
  - (a) FTS5 only
  - (b) Qdrant only
  - (c) hybrid + rerank
- **Compare:** Recall@10, nDCG@10.

If the hybrid+rerank numbers are not noticeably better than FTS5
alone on the actual data, **the architecture is wrong for this
corpus** and the problem is surfaced *now* instead of after six
phases of build-out.

**Output of this stage:** a number — "hybrid+rerank gets X%
Recall@10 vs Y% for lexical alone." That number is what justifies
every later phase.

### 2. The LLM plane with citations (Phase 2)

- Same pipeline, but feed top-K into Ollama, get a synthesized
  answer back.
- The prompt must require `[doc_id:chunk_id]` markers. Parse them.
  If the LLM cites a chunk that was *not* in the context, **reject
  the citation** and either drop the claim or re-roll.
- Read 20 of the answers manually. If the citations are fabricated
  or the answers are confidently wrong, the LLM plane is failing
  and there is a real signal to act on.

### 3. The Google Drive connector (Phase 4 lite)

- OAuth2 flow against the own account, scope = `drive.readonly`.
- Delta sync: Drive Changes API with a stored `startPageToken`.
  Cursors in a local SQLite file.
- Same `Normalize → Chunk → Embed → Write` pipeline. Drive files
  are just files with metadata.

---

## The minimum first commit that is shippable

```
CB/
├── docker-compose.yml         # Qdrant + (optional OpenSearch)
├── pyproject.toml             # poetry / pip
├── src/cb/
│   ├── api.py                 # FastAPI: /query, /ingest
│   ├── query/                 # normalize, router, retrieval, rerank
│   ├── indexing/              # connector, normalizer, chunker, embedder
│   ├── llm/                   # context builder, answer, citation verifier
│   ├── storage/               # Qdrant + FTS5 clients
│   └── eval/                  # golden set + runner
├── connectors/filesystem/     # walks C:/, D:/, respects .ragignore
├── eval/golden/queries.jsonl  # 50 hand-written queries with judged docs
├── data/                      # Qdrant storage, SQLite FTS5 db, blob store
└── README.md                  # how to run
```

One command — `docker compose up && uvicorn cb.api:app` — brings the
whole stack up. The first 30 queries run on it are the real test.

---

## The honest answer

**Don't test it by writing tests first. Test it by running 50 queries
against the own data and reading the answers.**

- If a search for *"vendor data retention"* finds the actual vendor
  contracts, the architecture works.
- If it returns a Jira ticket from 2021 about something unrelated,
  something is wrong, and the eval set tells you what.

**The eval harness is the test. Everything else is plumbing.**

---

## Plan to get test data

The corpus on the local PC (Documents, Repos, project folders) is
real but small. To stress-test retrieval with the kind of *volume*
an enterprise corpus has, synthetic data is needed. The strategy
below combines a Microsoft-provided sandbox with a seed-and-multiply
approach for the rest.

> **Note on Azure AI Foundry:** its "Synthetic Data" feature generates
> Q&A test pairs from documents you already have. It does **not**
> generate the source documents (the 500 fake PDF contracts or 1,000
> fake Jira tickets) needed to populate a Company Brain. The plan
> below is the way to actually create that source data.

### 1. Microsoft sources (SharePoint, Teams, Outlook) — use the M365 sandbox

Do not create these manually. Microsoft offers a free **Instant
Sandbox** designed exactly for this.

**Solution: Microsoft 365 Developer Program**

What you get:

- A pre-configured fake-company tenant (E5 license)
- 16 fictitious users (with photos and names)
- Pre-loaded data: inboxes full of sample emails, calendars with
  events, files in OneDrive and SharePoint
- Sample Teams chats and channels

How to use it: point the SharePoint / Outlook / Teams connectors at
this developer tenant. This immediately gives a valid, realistic
dataset for those three sources without writing a single file.

### 2. Other sources (Jira, Slack, Confluence, S3) — seed and multiply

No equivalent sandbox exists for these. Use an LLM (e.g. GPT-4o in
Azure Foundry, or a local Ollama model) to write the fake files, then
multiply with Python.

**The Seed & Multiply strategy:**

- You don't need 1,000 unique prompts.
- You need ~5 realistic templates, then a script that multiplies
  them with small variations (dates, names, statuses) to get volume.

**Step 1 — Generate the content** (in the LLM chat playground):

> *"Generate a JSON export of a Jira ticket for a 'Login Bug'. Include
> fields: key, summary, description, status, assignee, comments
> list."*

**Step 2 — Multiply with Python.** The script below generates
realistic fake Jira tickets and a fake Slack thread. It is a starter
template; extend it to produce 50–100 JSON files per source, then
upload them to a local folder (or Azure Blob / S3) and point the
RAG system at that container as if it were the live source.

```python
import json
import random
import datetime

def generate_fake_data():
    # 1. Fake Jira Tickets
    statuses = ["To Do", "In Progress", "Code Review", "Done"]
    priorities = ["Low", "Medium", "High", "Critical"]
    jira_tickets = []

    for i in range(1, 6):  # Generates 5 examples; bump to 100 for volume
        ticket = {
            "id": f"XY-{1000 + i}",
            "summary": f"Fix issue with {random.choice(['login', 'checkout', 'profile', 'search'])} page",
            "status": random.choice(statuses),
            "priority": random.choice(priorities),
            "assignee": f"User_{random.randint(1, 5)}",
            "created": (datetime.date.today() - datetime.timedelta(days=random.randint(1, 30))).isoformat(),
            "description": "User reported a bug when clicking the submit button. Error 500 returned."
        }
        jira_tickets.append(ticket)

    # 2. Fake Slack Thread
    slack_thread = {
        "channel": "#engineering-incidents",
        "messages": [
            {"user": "Alice", "text": "Has anyone seen the latency spike on the payment gateway?", "ts": "10:00 AM"},
            {"user": "Bob",   "text": "Checking NewRelic now. Looks like a database lock.",            "ts": "10:02 AM"},
            {"user": "Alice", "text": "Can we restart the replica?",                                   "ts": "10:05 AM"},
        ],
    }

    return jira_tickets, slack_thread


if __name__ == "__main__":
    jira_data, slack_data = generate_fake_data()
    print("--- FAKE JIRA TICKETS ---")
    print(json.dumps(jira_data, indent=2))
    print("\n--- FAKE SLACK THREAD ---")
    print(json.dumps(slack_data, indent=2))
```

### Recommendation (concrete next steps)

1. **Register for the M365 Developer Program** today — solves the
   Outlook / SharePoint / Teams data gap for free.
2. **Use the script above** (extended to ~50–100 variations per
   source) to generate JSON files for Jira, Slack, and Confluence.
3. **Point the connectors** at the M365 sandbox (for Microsoft
   sources) and at a local `data/fake-sources/` folder (for
   everything else) so the indexing plane has real volume to chew on
   before any production connector is built.

