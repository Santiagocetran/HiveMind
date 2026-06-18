# The First Consolidation Cycle, Explained

**What this is:** a plain-language account of what we actually did in the first manual memory-consolidation cycle (issue #1), what the resulting brain looks like, and what the moving parts (pages, links, chunks, embeddings, spend) actually *mean*. Written for a human reading later — not the agent.

**Date:** 2026-06-18 · **Host:** `mona` · **GBrain:** `v0.42.38.0`, PGLite, embeddings off.

**Companion docs:** [`explained.md`](explained.md) (what GBrain is in general), [`../loops/consolidation-write-policy.md`](../loops/consolidation-write-policy.md) (the rules we applied), [`../loops/loop-plan.md`](../loops/loop-plan.md) (where this fits the roadmap).

---

## 1. The one-paragraph version

Hermes (the agent on `mona`) had a messy `MEMORY.md` — 8 raw facts, several of them stale or duplicated. We turned that blob into **3 clean, typed, interlinked pages** living in a private, version-controlled markdown repo, then projected them into GBrain so they form a small **knowledge graph** an agent can search and traverse. Everything was done **for free** (no embeddings, no LLM calls, no money spent). This proves the pipeline end-to-end on real data before we automate it.

---

## 2. What we actually did

1. **Read the raw memory** — `~/.hermes/memories/MEMORY.md`, 8 `§`-delimited facts.
2. **Curated it** against the [write-policy](../loops/consolidation-write-policy.md):
   - **Dropped stale facts** — the `kate` identity, the `ia-bridge-mcp` project, and the `Kimi` model bug all described a *past* setup that no longer exists.
   - **Routed preferences to `USER.md`** — the "this is my machine / full autonomy" policy is behavior, not a graph fact, so it was rewritten into `USER.md` (scoped: free to operate the machine; pause only for spending money or writing the shared brain).
   - **Dropped ephemeral noise** — "user approved the diff" was true for one moment, never a durable fact.
   - **Kept the current, durable facts** — and rewrote them to reflect today's reality.
3. **Built a private brain repo** — `~/brain` on `mona`, a git repo (the *truth*), with 3 markdown pages. It is **never pushed** to the public HiveMind repo (personal brain content stays private).
4. **Projected it into GBrain** — `gbrain sync --repo ~/brain --no-embed`, then `gbrain extract links` to wire up the graph.
5. **Verified retrieval** — searched for known facts and walked the graph; everything returns the right page.

A genuine lesson surfaced along the way: GBrain only builds graph edges from **full-slug** wikilinks (`[[tools/hermes]]`), not bare ones (`[[hermes]]`), across directories. That bug kept the graph at half-strength until fixed — and is now a permanent rule in the write-policy.

---

## 3. The brain that exists now

**Three pages, four links:**

```
                    ┌─────────────────────┐
                    │   tools/hermes      │   "Hermes: agent on mona,
              ┌────▶│   (type: tool)      │◀────┐  DeepSeek, kawaii, profiles,
              │     └─────────────────────┘     │  markdown memory, MCP"
              │            ▲   │                 │
   "owns this │            │   │                 │ "installed on mona for…"
    machine"  │            │   │                 │
              │            │   ▼                 │
  ┌───────────────────────┐   ┌─────────────────────┐
  │ concepts/             │   │  tools/gbrain       │  "GBrain: relationships
  │ mona-environment      │   │  (type: tool)       │   layer, PGLite, no-spend,
  │ (type: note)          │   └─────────────────────┘   projection of markdown"
  └───────────────────────┘
   "mona is Hermes' own machine,
    NOPASSWD sudo, git = agent@mona.local"
```

Edges (all `link_type: mentions`, `source: markdown`):
- `tools/hermes` ↔ `tools/gbrain` (each references the other)
- `tools/hermes` ↔ `concepts/mona-environment` (each references the other)

`tools/hermes` is the **hub** — every other page connects through it. From it, `gbrain graph tools/hermes --depth 2` reaches the whole brain.

**Before → after:** 1 undifferentiated blob page, 0 links → **3 typed pages, 4 typed links.** That's the difference between a dumped note and a graph.

---

## 4. What "configuring GBrain in this Hermes" actually means

GBrain isn't bolted into Hermes' brain — it's a **separate service Hermes can reach**, deliberately isolated:

- **It's the "relationships" layer.** Hermes already had truth (markdown), recall (an FTS search DB), and preferences (`USER.md`). GBrain adds the missing layer: a **typed knowledge graph** — facts *and the connections between them* — that any agent can query.
- **It lives in its own sandbox.** GBrain's data is at `~/gbrain-data` (PGLite — a local database file, no server). It's wired to a dedicated **`gbrain` Hermes profile** over MCP, so the *default* Hermes personality, memory, and sessions are untouched. Turning GBrain on changed nothing about how normal Hermes behaves.
- **Markdown is still the boss.** The brain repo (`~/brain`) is the source of truth; GBrain is a **rebuildable projection** of it. If the database is ever corrupted or deleted, you re-run `sync` and get the brain back from the files. You can never lose a fact to GBrain.
- **It's the same brain other agents will share later.** The point of HiveMind is that Claude Code and Codex eventually read/write this *same* graph — so a fact Hermes learns is instantly available to all of them. This cycle is the first proof that the brain holds something worth sharing.

---

## 5. Glossary — what the numbers mean

When you run `gbrain stats` you see counts like `Pages: 3, Chunks: 5, Embedded: 0, Links: 4, Tags: 7`. Here's what each is:

| Term | What it means | In our brain |
|---|---|---|
| **Page** | One markdown file = one fact-cluster. The atomic unit of the brain. Its `type` (tool, note, project…) comes from its folder. | **3** (`hermes`, `gbrain`, `mona-environment`) |
| **Link (edge)** | A *typed relationship* between two pages, built automatically from `[[wikilinks]]`. This is "the graph" — the thing a search engine doesn't have. | **4** |
| **Chunk** | A sub-slice of a page, the unit actually indexed for search. A short page = 1 chunk. (Soft-deleted pages leave tombstone chunks for 72h, which is why the count can be higher than pages.) | **5** (3 real + 2 tombstones, auto-purging) |
| **Embedded** | How many chunks have **vector embeddings** — numeric representations that enable *semantic* ("means the same thing") search. We have these **off** (no embeddings key, and they cost money). | **0** (intentionally) |
| **Tag** | A label from a page's frontmatter, for filtering. | **7** |
| **Timeline** | Dated events attached to pages. We didn't add any. | **0** |

### Search: two kinds, and why we only use one

- **Lexical search** (`gbrain search`) — keyword/text matching. **Free, instant, no LLM.** It found every fact we tested ("DeepSeek" → the Hermes page, "owner autonomy sudo" → the environment page). This is all we use in Phase 1.
- **Semantic search** (`gbrain query`/`ask`, and embeddings) — matches *meaning*, not just words. More powerful, but needs embeddings (a paid API) and can call an LLM. **Deferred** until lexical search starts to hurt.

### Spend: what costs money, and why this cost nothing

GBrain has two free tiers and one paid tier:

- **Free & instant:** capturing pages, `sync`, **auto-linking / `extract links`** (pure pattern-matching on wikilinks — no LLM), lexical `search`, graph traversal. *Everything in this cycle.*
- **Paid (LLM/embeddings):** vector `embed`, semantic `query`/`ask` synthesis, and the nightly **`dream` cycle** (which dedupes, fixes citations, finds contradictions using an LLM). **All deliberately off.** The guardrail: a loop that spends money unattended is a loop that makes mistakes unattended.

So this cycle's total cost was **$0** — by design, and verified (the only thing that *would* have spent, embedding during `sync`, refused to run without a key rather than charging us).

---

## 6. What this does — and doesn't — do yet

**Does:** gives Hermes a small, clean, queryable knowledge graph of current facts about itself and its machine; proves the curate → project → retrieve pipeline on real data; defines a reusable write-policy.

**Doesn't yet:**
- **No semantic search** — lexical only, until an embeddings key + approved spend (Phase 2).
- **No automation** — this was a *manual* cycle. The `memory_monitor` loop is still off. Next step is encoding the [write-policy](../loops/consolidation-write-policy.md) as a skill and flipping the loop to **propose-only**.
- **No dream cycle** — no unattended LLM maintenance.
- **One machine, one agent** — the cross-agent shared brain (Claude Code + Codex on the same graph) needs remote MCP, and comes later.

The plumbing is proven and safe. The value grows from here — more curated facts, then automation, then sharing.
