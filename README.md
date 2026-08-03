# Company Brain (CB)

Enterprise retrieval-augmented system that acts as the **organizational
memory** of a company. Indexes every document, email, ticket, wiki page,
code repo, and chat the company has, and lets any authorized employee ask
natural-language questions and get a **cited answer** in <2 seconds.

## What's in v1 (planned)

- Hybrid retrieval: BM25 (OpenSearch) + vectors (Qdrant) + cross-encoder rerank
- Per-source connectors: Drive, Confluence, Slack, Jira, GitHub, Gmail, Salesforce
- LLM synthesis with verified inline citations
- First-class ACL enforcement at the retrieval layer
- Multi-tenant SaaS + on-prem (Helm) deployment
- Continuous eval harness with a private golden set

## Status

Architecture-only repo as of this commit. No code yet.

See [`Architecture.md`](./Architecture.md) for the full design and the
relationship to the `LocalRAG1` project.

## Quick sketch

```
Query → normalize → ACL resolve → router ─┬─→ cheap path (lexical/vector only)
                                           ├─→ deep path (hybrid + rerank + LLM)
                                           └─→ agent path (multi-hop)

Indexing:  Sources → Connectors → Normalize → Chunk → Embed → Multi-index write
                                                              ├─ OpenSearch (BM25)
                                                              ├─ Qdrant (vectors)
                                                              ├─ Neo4j (graph, Phase 6)
                                                              └─ Blob store (originals)
```

## Why this exists alongside LocalRAG1

| | LocalRAG1 | Company Brain |
|---|---|---|
| Scale | 1 user, 1 folder | N users, M sources, 1 org |
| Retrieval | Lexical only | Hybrid + rerank |
| LLM | Opt-in fallback | Default renderer |
| Permissions | n/a | First-class |
| Deploy | Single .exe | Kubernetes / on-prem |

LocalRAG1 is the *kernel* (personal, lightweight, no model). CB is the
*operating system* (enterprise, multi-tenant, permissioned). They are
complementary, not competing.
