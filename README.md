# HiveMind — a shared memory layer for AI agents, and the loops that use it

**Status:** Phase 1 complete — GBrain installed and verified on the Hermes host. Next: the first *loop*.
**Last updated:** 2026-06-10.

This repo is the design record and working notes for giving a multi-agent setup a **shared, persistent memory** — and then driving that memory with **loops** instead of manual prompting. It's part research, part engineering log.

---

## 1. The idea in one paragraph

AI agents are stateless strangers: every new session and every handoff between tools (Claude Code, Codex, Hermes, …) starts from zero, so you re-explain context endlessly. The **Hive Mind** is a single, user-owned memory layer that all agents read from and write to — so a thing learned by one agent is instantly available to all. We chose **GBrain** as the first concrete "brain," running alongside the **Hermes** agent, and we use **loops** (automations that find work, do it, check it, and record state) as the model for how an agent actually *interacts* with that brain over time.

Three threads, one system:
- **Theory** → the Hive Mind (why shared memory matters, and what's actually hard about it).
- **The brain** → GBrain (the storage + knowledge-graph + retrieval layer).
- **The interaction model** → loops (how an agent uses and maintains the brain without you in the chair).

---

## 2. Foundational theory — the Hive Mind

**The problem.** Each agent is an "isolated skull." Repos and markdown capture the *destination* (final decisions) but not the *journey* (the reasoning, false starts, and context that later become relevant). Multiply that across tools and machines and you get constant re-derivation — a tax that, unlike with humans, is technically avoidable.

**The vision.** One coherent "you" across all tools: a persistent, searchable, model-agnostic memory layer underneath every agent.

**What's actually hard (our key finding).** Storage is solved. Retrieval is mostly solved. The unsolved part is the **write/curation policy** — *what deserves to be remembered, by whom, when?* Dump everything and you get noise and worse recall; remember too little and the brain is useless. So the differentiator between a useful hive mind and an expensive liability is **curation discipline**, not the choice of database.

**The organizing principle: "Files = truth."** Canonical truth lives in inspectable, versioned **markdown files**. The knowledge graph and vector index are **derived, disposable projections** you can rebuild. This makes the system debuggable, portable, and vendor-neutral — and it's exactly how GBrain is built.

**The layered memory stack** (separation of concerns):

| Layer | Job | Tool in our setup |
|---|---|---|
| **Truth** | canonical artifacts/facts | markdown (git) |
| **Recall** | fast search over the corpus | keyword/FTS + (later) vectors |
| **Relationships** | entities & typed connections, *curated* | **GBrain** |
| **Continuity** | preferences, user model | Honcho / `USER.md` |
| **Episodic / journey** | replayable session history | (deferred — e.g. CASS) |
| **Governance** | turning memory into tracked work | the loop |

> See `hivemind-analysis.md` for the full critical analysis and `hivemind-first-plan.md` for the original (uncritical) plan it corrects.

---

## 3. The brain — why GBrain, and what it is

**Why GBrain first.** It's markdown-first ("Files = truth" by design), runs locally with zero server (PGLite), exposes ~89 tools over **MCP** (so any agent can use it), self-wires a typed knowledge graph from your notes, and ships an overnight maintenance **dream cycle** that dedupes, fixes citations, and finds contradictions. It's the "relationships + curated recall" layer of the stack.

**What it does.** Turns markdown pages into a **typed knowledge graph** (entities + relationships) plus a searchable index, then *reads and reasons* over it — not just ranked chunks, but synthesized, cited answers with gap analysis. "Search finds the pages; the brain reads them for you and writes the answer."

**The lifecycle of a thought:** `capture → signal-detection → auto-link (free, no LLM) → dream-cycle enrichment (LLM) → retrieval`.

> Full explainer — data model, retrieval engine, dream cycle, full-potential features, and how it differs from plain RAG/Obsidian — in `gbrain-explained.md`.

---

## 4. The interaction model — loops

**Loop engineering** (after Addy Osmani): instead of prompting an agent turn-by-turn, you build a small system that **finds the work, hands it out, checks it, records what's done, and decides the next thing**. You stop being the runtime and become its architect.

A loop's anatomy: **automation** (the heartbeat/schedule) · **skills** (persistent knowledge) · **connectors/MCP** (reach into real tools) · **maker + checker sub-agents** (split doing from verifying) · **state/memory on disk** (what's done + next).

**Why this is the right interaction model for the Hive Mind:** memory *is* the loop's spine. GBrain's dream cycle is literally a loop. The way Hermes should "use" the brain is: on a schedule, **read recent work → extract durable facts → verify them → write the good ones to the brain → record progress**. "Search before acting" becomes every agent's first move.

**The first loop we'll build — the memory-consolidation loop:**
```
discover recent session deltas
  → MAKER sub-agent extracts candidate durable facts
  → CHECKER sub-agent rejects noise / merges duplicates / flags contradictions
  → write survivors as clean pages into the brain
  → advance a watermark; repeat
```
Its defining choice: **the checker runs on the writes**, because curation is the hard problem (§2). Its checker maps onto the audit policy the operator already trusts.

> Full design and phasing in `hivemind-loop-plan.md`.

---

## 5. Current state (verified, 2026-06-10)

**Hermes** runs on host `mona`, with its own markdown memory (`MEMORY.md`/`USER.md`), an idle cron heartbeat, and an FTS recall DB — i.e. the loop surface already exists.

**GBrain Phase 1 — installed, isolated, and independently verified:**
- `gbrain v0.42.38.0`, PGLite brain, wired into a **dedicated `gbrain` Hermes profile** over MCP — the default Hermes personality is untouched.
- Cost-controlled and no-spend: `conservative` mode, embeddings **off**, dream cycle **deferred**.
- A security issue (an API key was hard-coded during install) was caught, the key **rotated**, and the config converted to a single-source env **reference** — verified resolving correctly.
- Robustness fixes applied: `GBRAIN_HOME` persisted, the empty default-brain trap removed.

**Honest assessment of where the brain is:**
- The knowledge graph is currently **one page, zero links** — `MEMORY.md` was imported as a single blob. This isn't a bug; it's the central lesson: **GBrain needs discrete, well-formed pages to build relationships.** Dumping raw memory gives it nothing to connect.
- Semantic search, synthesis, and the dream cycle are **gated** behind (a) an embeddings provider key and (b) approving LLM spend — both deliberate.

> Net: the *plumbing* is correct and safe. The *value* is still ahead, and it's unlocked by **structured input + curation**, not more configuration.

---

## 6. Key insights (the things worth sharing)

1. **The hard problem is curation, not storage.** Most of the effort and most of the risk live in the write policy.
2. **Files = truth.** Anchor everything on inspectable markdown; treat graphs/vectors as rebuildable projections.
3. **Memory and loops are one system.** A brain with no loop feeding it stays empty; a loop with no brain forgets everything. GBrain's dream cycle proves the pattern.
4. **Garbage in → empty graph out.** Verified live: raw dumps don't build relationships. The consolidation loop is what makes the brain actually useful.
5. **Stay the engineer.** Loops that run unattended also make mistakes unattended — hence maker/checker, and reading what the loop writes.

---

## 7. Roadmap

| Phase | What | State |
|---|---|---|
| Theory & analysis | Hive Mind problem, layered stack, critique | ✅ done |
| GBrain Phase 1 | install, isolate, secure, verify | ✅ done |
| Revoke old key | operator action on DeepSeek dashboard | ⏳ pending |
| **Memory-consolidation loop** | first real loop; produces clean pages, fixes the empty-graph problem | ▶️ **next** |
| GBrain Phase 2 | embeddings key → semantic search; enable dream cycle | later |
| Cross-machine | multi-source federation (real Postgres) for a true shared brain | later |

---

## 8. Documents in this repo

- **`README.md`** — this overview.
- **`hivemind-first-plan.md`** — the original plan (Grok-generated; directionally right, factually imperfect).
- **`hivemind-analysis.md`** — critical analysis that corrects it; introduces "Files = truth" and the layered stack.
- **`hivemind-loop-plan.md`** — loop engineering applied to Hermes; the memory-consolidation loop design.
- **`gbrain-install-guide.md`** — the agent-executable install runbook (no-spend, isolated, audited).
- **`gbrain-explained.md`** — deep explainer of GBrain: how it works and how to get its full potential.
