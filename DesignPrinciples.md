# Design Principles — Company Brain (CB)

> The principles the **code itself** must follow. This is the
> companion to `Architecture.md` (which describes *what* the system
> is) and `SystemInstruction.md` (which describes *how we work in a
> session*). When you are writing a module and the question is
> "should I structure it like X or Y?", read this file.

---

## 1. Pluggability — the hard rule for every subsystem

**Every subsystem with more than one realistic implementation in
production must be configurable, modular, and pluggable.** This is
a hard constraint, not a nice-to-have. Treat it the way the
architecture treats ACL: a first-class concern.

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

### Concrete examples (the user-named ones and the rest)

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

### Acceptance test for any new code

Before considering a new module done, mentally check:

1. Could a second implementation of this be added without changing
   the caller? (If no, refactor to a Protocol first.)
2. Is the concrete client import hidden inside the implementation
   file? (If not, move it.)
3. Is selection driven by a single config key? (If no, the design
   is wrong.)
4. Could a unit test substitute a fake implementation by
   registering it? (If no, the Protocol is too narrow or too wide.)

If any answer is no, the code is not done. Fix it before
considering it ready for review.

---

## 2. Where to look next

- `Architecture.md` — the system design, the four planes, the
  retrieval pipeline, the latency budget, the security model.
  Principle #8 in `Architecture.md` cross-references this file.
- `SystemInstruction.md` — session conduct, commit hygiene, answer
  style. Not design rules.
- `STATUS.md` — where the project is right now, what's next.
- `Discussion1.md` — the single-PC test plan and the synthetic-data
  strategy.
