# HiveMind — a shared memory layer for AI agents, and the loops that use it

This repo is the design record for giving a multi-agent setup a **shared, persistent memory** — and then driving that memory with **loops** instead of manual prompting. Part research, part engineering log.

## The idea in one paragraph

AI agents are stateless strangers: every new session and every handoff between tools (Claude Code, Codex, Hermes, …) starts from zero, so you re-explain context endlessly. The **Hive Mind** is a single, user-owned memory layer that all agents read from and write to — so a thing learned by one agent is instantly available to all. We use **GBrain** as the first concrete "brain," running alongside the **Hermes** agent, and **loops** (automations that find work, do it, check it, and record state) as the model for how an agent actually *interacts* with that brain over time.

Three threads, one system:
- **Theory** → the Hive Mind (why shared memory matters, and what's actually hard about it).
- **The brain** → GBrain (the storage + knowledge-graph + retrieval layer).
- **The interaction model** → loops (how an agent uses and maintains the brain without you in the chair).

---

## Foundational theory — the Hive Mind

**The problem.** Each agent is an "isolated skull." Repos and markdown capture the *destination* (final decisions) but not the *journey* (the reasoning, false starts, and context that later become relevant). Multiply that across tools and machines and you get constant re-derivation — a tax that, unlike with humans, is technically avoidable.

**The vision.** One coherent "you" across all tools: a persistent, searchable, model-agnostic memory layer underneath every agent.

**What's actually hard (the key finding).** Storage is solved. Retrieval is mostly solved. The unsolved part is the **write/curation policy** — *what deserves to be remembered, by whom, when?* Dump everything and you get noise and worse recall; remember too little and the brain is useless. The differentiator between a useful hive mind and an expensive liability is **curation discipline**, not the choice of database.

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

> Full critical analysis: [`hivemind-analysis.md`](hivemind-analysis.md). The original (uncritical) plan it corrects is kept at [`archive/hivemind-first-plan.md`](archive/hivemind-first-plan.md).

---

## The brain — GBrain

**Why GBrain.** It's markdown-first ("Files = truth" by design), runs locally with zero server (PGLite), exposes its tools over **MCP** (so any agent can use it), self-wires a typed knowledge graph from your notes, and ships an overnight maintenance **dream cycle** that dedupes, fixes citations, and finds contradictions. It's the "relationships + curated recall" layer of the stack.

**What it does.** Turns markdown pages into a **typed knowledge graph** (entities + relationships) plus a searchable index, then *reads and reasons* over it — not just ranked chunks, but synthesized, cited answers with gap analysis. "Search finds the pages; the brain reads them for you and writes the answer."

**The lifecycle of a thought:** `capture → signal-detection → auto-link (free, no LLM) → dream-cycle enrichment (LLM) → retrieval`.

> Deep explainer — data model, retrieval engine, dream cycle, full-potential features, RAG/Obsidian comparison — in [`gbrain/explained.md`](gbrain/explained.md). Setup runbook in [`gbrain/install-guide.md`](gbrain/install-guide.md).

---

## The interaction model — loops

**Loop engineering** (after Addy Osmani): instead of prompting an agent turn-by-turn, you build a small system that **finds the work, hands it out, checks it, records what's done, and decides the next thing**. You stop being the runtime and become its architect.

A loop's anatomy: **automation** (the heartbeat/schedule) · **skills** (persistent knowledge) · **connectors/MCP** (reach into real tools) · **maker + checker sub-agents** (split doing from verifying) · **state/memory on disk** (what's done + next).

**Why loops are the right interaction model for the Hive Mind:** memory *is* the loop's spine. GBrain's dream cycle is literally a loop. The way an agent should "use" the brain is: on a schedule, **read recent work → extract durable facts → verify them → write the good ones to the brain → record progress**. "Search before acting" becomes every agent's first move.

**The first loop — the memory-consolidation loop:**
```
discover recent session deltas
  → MAKER sub-agent extracts candidate durable facts
  → CHECKER sub-agent rejects noise / merges duplicates / flags contradictions
  → write survivors as clean pages into the brain
  → advance a watermark; repeat
```
Its defining choice: **the checker runs on the writes**, because curation is the hard problem.

> Full design and phasing in [`loops/loop-plan.md`](loops/loop-plan.md).

---

## Key insights

1. **The hard problem is curation, not storage.** Most of the effort and most of the risk live in the write policy.
2. **Files = truth.** Anchor everything on inspectable markdown; treat graphs/vectors as rebuildable projections.
3. **Memory and loops are one system.** A brain with no loop feeding it stays empty; a loop with no brain forgets everything.
4. **Garbage in → empty graph out.** Raw dumps don't build relationships; structured, curated input does.
5. **Stay the engineer.** Loops that run unattended also make mistakes unattended — hence maker/checker, and reading what the loop writes.

---

## Repo map

```
README.md              ← you are here (stable overview)
hivemind-analysis.md   ← foundational theory (the main hivemind doc)
STATUS.md              ← current state + roadmap (the living, evolving doc)
gbrain/
  explained.md         ← what GBrain is and how to get its full potential
  install-guide.md     ← agent-executable install runbook (no-spend, isolated)
loops/
  loop-plan.md         ← loop engineering applied to Hermes; the consolidation loop
archive/
  hivemind-first-plan.md ← the original Grok plan (kept for provenance)
```

---

**Current status & roadmap → [STATUS.md](STATUS.md)**
