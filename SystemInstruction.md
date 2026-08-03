# System Instruction — Company Brain (CB)

> Standing instructions for the AI assistant working on this repo.
> This file is **project-internal protocol**, not user-facing
> documentation. Read it at the start of every session before
> doing anything else.

---

## 1. Working agreement (set on the day the repo was first pushed)

1. **The assistant does NOT commit unless the user explicitly says
   "commit" (or equivalent: "commit it", "go ahead and commit",
   "yes, commit", "ship it", "land it") in the same turn.** Stage
   changes and show a diff; wait for the word. This is non-negotiable
   and overrides anything else in this file.
2. **The assistant NEVER pushes.** Full stop. The user owns the push
   button regardless of how the commit got there.
3. **All edits happen in the local working copy** `C:/repos/CB`.
4. The user verifies each commit before pushing. This is a
   deliberate trust/safety boundary, not a workaround for missing
   credentials.
5. Cached Git auth on this machine would technically let the
   assistant push. **That capability is not used.** Respect the
   agreement.

### Why this split exists

- The user wants to eyeball every commit and every push before they
  leave the machine. Two separate gates, two separate verbs:
  - **"commit"** = the user has reviewed the staged changes and
    approved them.
  - **"push"** = the user has reviewed the commit and approved it
    leaving the machine.
- Never collapse these. If the user says "push it" they are
  approving a push of whatever is already committed — they are NOT
  also implicitly approving a fresh commit.

### Common phrases that DO NOT count as a commit instruction

- "go ahead" / "do it" / "make it so" / "looks good" → these refer
  to the *current action being discussed*, not to committing.
  If in doubt, ask.
- "commit when you're done" → counts as a commit instruction,
  IF the user is clearly talking about a specific, already-discussed
  unit of work. If the scope is ambiguous, stage and ask.
- "land it", "ship it", "LGTM commit it" → counts.

### Default behaviour when uncertain

Stage the changes, show a concise diff summary (file list + 1 line
per file), and ask: *"Ready to commit? (say the word)"*. One line.
Do not commit. Do not push.

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

## 10. Where the design rules live

This file is for **how we work** (commit hygiene, answer style,
session protocol). Design rules — the principles the code itself
must follow — live in their own files so the right reader finds
them:

- **Pluggability** (subsystem modularity, registry pattern, config-
  driven selection, anti-patterns, acceptance test for new code):
  see [`DesignPrinciples.md`](./DesignPrinciples.md).

When in doubt about *how to write a module*, read
`DesignPrinciples.md`. When in doubt about *how to behave in a
session*, read this file.

