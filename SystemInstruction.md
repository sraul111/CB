# System Instruction — Company Brain (CB)

> Standing instructions for the AI assistant working on this repo.
> This file is **project-internal protocol**, not user-facing
> documentation. Read it at the start of every session before
> doing anything else.

---

## 1. Working agreement (set on the day the repo was first pushed)

1. **All edits happen in the local working copy** `C:/repos/CB`.
2. **I (the assistant) commit changes but do NOT push.**
3. **The user pushes manually** after verifying the commit.
4. The user verifies each commit before pushing. The user owns the
   push button. Do not push from my side under any circumstance
   unless the user explicitly says "push it" in that same turn.

### Why this split exists

- The user wants to eyeball every commit before it leaves their
  machine. This is a deliberate trust/safety boundary, not a
  workaround for missing credentials.
- Cached Git auth is available on this machine and would technically
  let me push. **Do not use that capability.** Respect the agreement.

---

## 2. Commit format

Use this format for every commit:

```
<imperative one-line summary, ≤ 72 chars>

- Bullet describing the change
- Bullet describing the change
- Bullet describing the change (if any)

[optional] Co-authored-by: <...> if the user contributed substantially
```

- Commit as the user, not as the assistant. The git identity on this
  machine is already `Sujit Kumar Raul <sujitkumar_raul@epam.com>`.
  If a global config exists, use it. If not, pass `-c user.name=...`
  `-c user.email=...` to the commit command.
- **Never** amend, force-push, or rewrite published history.

---

## 3. Response style — crisp answers

- **Be concise.** The user prefers short, direct answers.
- Prefer **bullet points and tables** over long prose.
- **Code first, explanation second** when the user is asking "how
  do I do X" or "show me Y". One sentence of context, then the code.
- **Don't restate the question.** Don't recap what the user just
  said. Answer.
- **No filler** ("Great question!", "Sure, I can help with that!",
  "Let me explain..."). Get to the point.
- **When uncertain, say so in one line**, then give the best
  informed guess. Don't pad uncertainty with three paragraphs of
  caveats.
- **Tables and code blocks are preferred** to running paragraphs.
  Use them liberally.
- **Default answer length: as short as possible while still
  complete.** If the user wants more, they will ask.

---

## 4. Project-specific conventions (current as of first push)

- **Language**: Python 3.11+ with FastAPI. (Decision locked in.
  Revisit only if API tier becomes the measured bottleneck.)
- **Stack on one PC**: Qdrant (Docker), SQLite FTS5 for v0 lexical
  (OpenSearch when >100k docs), `bge-large-en-v1.5` embeddings,
  `bge-reranker-v2-m3` reranker, Ollama for the answer LLM.
- **Single-tenant for Phase 1.** Multi-tenancy is Phase 3 work.
- **ACL is deferred to Phase 3.** Treat the corpus as world-readable
  in Phase 1 / 2.
- **Phase 6 (Neo4j graph) is deferred indefinitely** until a
  concrete relational query need appears.
- **Repo is private on GitHub** under `sraul111/CB`. Remote URL:
  `https://github.com/sraul111/CB.git`. Branch: `main`.

See `Architecture.md` for the full design, `STATUS.md` for where we
are, and `Discussion1.md` for the single-PC test plan.

---

## 5. Session-start checklist (for the assistant, every session)

Before doing anything else, run:

```bash
cd C:/repos/CB && git status && git log --oneline -5
```

If the working tree is dirty, surface that to the user immediately.
If there are unpushed commits, mention them. Do not assume the
prior session's state.

---

## 6. What the assistant must NEVER do

- ❌ Push to `origin` (or any remote) without explicit per-turn
  permission from the user.
- ❌ Force-push, amend published commits, or rewrite history.
- ❌ Create, delete, or rename GitHub repos via API/CLI.
- ❌ Modify global git config (`--global`).
- ❌ Run `git config --get-all` on secrets or print credential
  material.
- ❌ Edit files outside `C:/repos/CB` unless the user asks.
- ❌ Run commands that require admin/UAC elevation that hasn't
  already been granted to this shell.

---

## 7. What the assistant SHOULD do

- ✅ Read `STATUS.md` and `Discussion1.md` at the start of every
  session if there's been a break — the user's "where are we" file
  is `STATUS.md`.
- ✅ Stage and commit clearly-scoped changes.
- ✅ Show the user the diff (or at least the file list + a one-line
  summary per file) before committing, when the change is
  non-trivial.
- ✅ When in doubt about a design decision, surface the question
  rather than guessing. One-line question, two-line trade-off,
  one-line recommendation.
- ✅ Keep `STATUS.md` updated at the end of any session that
  changed the project's next step.

---

## 8. End-of-day handoff

When the user says they're done for the day (e.g. "my day ends
now"), the assistant should:

1. Make sure all in-progress work is either committed or
   explicitly stashed / noted as uncommitted in `STATUS.md`.
2. Update `STATUS.md` with: what got done today, what's pending,
   and what the first thing to do next session is.
3. **Do not push.** Leave commits local for the user to push.
4. Give a 3–5 bullet handoff summary, crisp, in chat.

---

## 9. Resuming after a break

Quick-start for any future session:

```bash
cd C:/repos/CB
cat SystemInstruction.md     # this file
cat STATUS.md                # where we are
git log --oneline -10        # what's been done
git status                   # any uncommitted work
```

Then ask the user "what's next" and stop.

---

## 10. Pluggability principle (non-negotiable for all code)

**The user requires every subsystem to be configurable, modular, and
pluggable.** This is a hard constraint on every line of code, not a
nice-to-have. Treat it the way the architecture treats ACL: a
first-class concern.

### What "pluggable" means in practice

Every subsystem that has more than one reasonable implementation in
production must be:

1. **Behind a thin interface** that the rest of the system depends
   on, not the concrete class.
2. **Selected by configuration**, not by `if/else` in business code.
3. **Addable without touching the callers.** Drop a new module
   file, register it, change the config value. No edits to
   retrieval, query, or LLM code.

### The pattern to follow

For every pluggable subsystem, use this structure:

```
src/cb/<plane>/<subsystem>/
    __init__.py
    base.py            # the Protocol / ABC the rest of the system depends on
    registry.py        # the single place that maps a config string → class
    fts5.py            # one implementation
    bm25_opensearch.py # another implementation
    ...
```

- `base.py` defines the **interface** (Python `Protocol` preferred
  over ABC for structural typing — fewer imports, easier mocking).
- `registry.py` is a `@register("name")` decorator + a
  `get(name: str) -> SomeProtocol` factory. One registry per
  subsystem.
- Each implementation file is **self-contained**: imports its own
  client library, no cross-implementation coupling.
- The config file picks the implementation by name. Example:

  ```yaml
  retrieval:
    lexical: fts5          # or: bm25_opensearch, bm25_elasticsearch
    vector: qdrant         # or: milvus, pgvector
    reranker: bge_local    # or: cohere_rerank3
    graph: neo4j           # or: memgraph, none
    fusion: rrf            # or: weighted_sum
  indexing:
    sources:
      - filesystem
      - slack
      - azure_devops       # not jira? swap it in config
    chunker:
      markdown: heading_bounded
      code: ast_aware
  llm:
    answer: ollama_qwen    # or: claude_sonnet, openai_gpt4o, llama_70b
  ```

  Changing one line swaps an entire subsystem. No code edits.

### Concrete examples the user named

| Subsystem | Pluggable on | Why |
|---|---|---|
| Lexical index | `FTS5` ↔ `BM25 (OpenSearch)` ↔ `BM25 (Elasticsearch)` ↔ anything else | Local dev vs enterprise scale, vendor preference, air-gapped deploys |
| Input sources | `slack` ↔ `azure_devops` ↔ `teams` ↔ `mattermost` etc. | No two orgs have the same tool stack. Adding a new source is one new file + one config line. |
| Graph index | `neo4j` ↔ `memgraph` ↔ `arangodb` ↔ `none` | Graph tech is a deployment preference, not a hard requirement. Some orgs won't run Neo4j. |
| Vector index | `qdrant` ↔ `milvus` ↔ `pgvector` | Same — vendor choice, scale, ops comfort. |
| Embedder | `bge_local` ↔ `voyage_api` ↔ `openai_embeddings` | Cost, air-gap, multilingual needs. |
| Reranker | `bge_local` ↔ `cohere_rerank3` ↔ `colbert` | Quality vs cost vs ops. |
| LLM | `ollama_qwen` ↔ `claude_sonnet` ↔ `openai_gpt4o` ↔ `llama_70b` | Cloud vs on-prem, cost, capability. |
| Citation verifier | `substring` ↔ `entailment_model` | Cheap vs accurate. |
| Object store | `local_fs` ↔ `s3` ↔ `azure_blob` ↔ `gcs` | Deployment model. |
| Identity / ACL | `okta` ↔ `azure_ad` ↔ `none` (single-user) | Every org has a different IdP. |

### Anti-patterns to avoid

- ❌ `if config.lexical == "fts5": ... else: ...` in retrieval code.
  That's a registry smell. The retrieval code should call
  `LexicalIndex.get(config.lexical).search(...)` and not know what
  `fts5` is.
- ❌ Hard-coded `import qdrant_client` at the top of retrieval code.
  Imports of the concrete client must live inside the
  implementation file, lazy-loaded.
- ❌ A "kitchen sink" interface that forces every implementation to
  support features it doesn't need. The protocol should be the
  **minimum** surface every implementation can satisfy. Extra
  capabilities are opt-in via `isinstance` / `hasattr` checks against
  a capability protocol, not the base.
- ❌ Magic strings in business code. The only place a string like
  `"fts5"` should appear is the config file and the registration
  decorator. Everywhere else, the value is typed (a `Literal` or an
  enum, but the **enum value is what the registry looks up**, not
  used for `if/else`).
- ❌ Config values that require code to interpret in more than one
  place. The config schema is the contract.

### What this does NOT mean

- **Not over-abstracted.** If a subsystem has exactly one
  implementation in production and no realistic second one in the
  next 12 months, do NOT make it pluggable. Premature pluggability
  is its own cost. Examples of subsystems that are NOT pluggable
  in Phase 1: the query normalizer, the context builder, the
  eval harness runner. They're internal; one implementation.
- **Not a plugin marketplace.** Pluggable here means "swappable
  inside this codebase," not "third-party extensions." We are not
  building LangChain.
- **Not a config-only system.** A new implementation still requires
  writing the implementation file. The config picks which one runs;
  it does not generate new code.

### How this maps to the four planes

| Plane | Pluggable subsystems |
|---|---|
| **Query** | Identity provider (for ACL), query normalizer language detector (if multilingual) |
| **Retrieval** | Lexical index, vector index, graph index, reranker, fusion algorithm, embedder |
| **LLM** | Answer LLM, citation verifier, context-window strategy |
| **Indexing** | Every source connector, per-format chunker, object store, embedder (shared with retrieval) |

Every entry in this table follows the protocol + registry pattern
in `src/cb/<plane>/<subsystem>/`.

### Acceptance test for any new code I write

Before committing any new module, mentally check:

1. Could a second implementation of this be added without changing
   the caller? (If no, refactor to a Protocol first.)
2. Is the concrete client import hidden inside the implementation
   file? (If not, move it.)
3. Is selection driven by a single config key? (If no, the design
   is wrong.)
4. Could a unit test substitute a fake implementation by
   registering it? (If no, the Protocol is too narrow or too wide.)

If any answer is no, the code is not done. Fix it before
committing.

