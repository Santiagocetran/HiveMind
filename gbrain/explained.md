# GBrain, Explained — what it is, how it works, and how to get its full potential

**Author:** Claude Code (HiveMind project), 2026-06-10.
**Grounded in:** GBrain README/architecture + your live install on `mona` (`gbrain v0.42.38.0`, schema pack `gbrain-base-v2`, 89 MCP tools, PGLite, conservative mode, embeddings off).
**Companion docs:** `install-guide.md` (setup), `../loops/loop-plan.md` (where GBrain fits the stack).

---

## 0. The one-sentence version

> "Search finds the pages. The brain reads them for you and writes the answer. This is the difference between a search engine and a brain."

GBrain is a **personal knowledge brain**: markdown files are the source of truth, and GBrain continuously turns them into a *typed knowledge graph + searchable index*, then exposes ~89 tools over MCP so any agent (Hermes, Claude Code, Codex) can capture into it and reason over it. It is the **"relationships + curated recall" layer** of your HiveMind stack.

---

## 1. The core philosophy: **Files = truth**

This is the single most important idea, and it's why GBrain is trustworthy:

- **Your knowledge lives as plain markdown in a git repo** ("the brain repo"). Human-readable, diffable, portable, vendor-neutral.
- **Postgres (PGLite or server) is a *derived index*** — GBrain syncs the repo into it for fast retrieval. The graph edges and vector embeddings are **disposable projections** of the markdown. If the index drifts or rots, you rebuild it from the files.
- **Deletes in git become soft-deletes in the DB**; re-syncing short-circuits on content-hash match (cheap, idempotent).

> Practical consequence: you never lose data to GBrain, and you can always inspect/edit the truth by hand. This is the CQRS / "source-of-truth + read-models" pattern applied to memory.

---

## 2. The data model — what's actually in the brain

| Concept | What it is | How it's created |
|---|---|---|
| **Page** | A markdown file with frontmatter + body. The atomic unit. Has a `type` (note, person, company, concept, daily…) inferred from its path prefix (`people/`, `concepts/`, …). | `capture`, `put`, `import`, `sync` |
| **Chunk** | A semantic sub-unit of a page, the thing actually indexed for retrieval. | Auto on write/embed |
| **Typed edge (link)** | A relationship between pages with a *type* and *provenance* — `works_at`, `founded`, `invested_in`, `advises`, etc. This is the "graph." | Auto-link (pattern matching, **no LLM**) + `extract links` + manual `link` |
| **Tag** | Frontmatter or auto-tag label. Accumulates across enrichment (never silently deleted). | Frontmatter, signal detector, dream |
| **Timeline** | Dated events attached to a page (creation, modification, manual entries). | Auto + `timeline-add` |
| **Salience** | A query-independent "how much does this matter" score (emotional weight + activity). | Dream cycle (`salience`) |

The graph is the differentiator: **multi-hop traversal reaches answers pure vector search can't** (GBrain claims +31.4 P@5 over vector-only). You can walk it directly: `gbrain graph <slug> --depth N`, `gbrain graph-query <slug> --type works_at --direction out`, `gbrain backlinks <slug>`.

---

## 3. How it works — the lifecycle of a thought

```
CAPTURE → INBOX → ENRICHMENT (dream) → GRAPH → RETRIEVAL
```

1. **Capture** — one entrypoint: `gbrain capture "thought"` (or `--file`, `--stdin`, webhook `/ingest`, mobile drop-folder). Lands at `inbox/YYYY-MM-DD-<hash>` with a returned slug. Synchronous receipt.
2. **Signal detection** — runs on every inbound message: pulls out entities, names, links, time-sensitive todos. (Lightweight.)
3. **Auto-link** — fires on every page write: turns `[[wiki/people/bob]]`-style references into typed edges. **Zero LLM calls — pure pattern matching.** The graph self-wires for free.
4. **Enrichment (the dream cycle)** — scheduled, heavier, LLM-backed (see §5).
5. **Retrieval** — `search` / `query` / graph traversal / salience (see §4).

The split matters: **steps 2–3 are free and instant; step 4 is where the LLM spend and the real "intelligence" lives.**

---

## 4. The retrieval engine — `search` vs `query`, and the scoring stack

Two retrieval verbs, very different cost/behavior:

| Verb | What it does | LLM? | Use when |
|---|---|---|---|
| `gbrain search <q>` | Keyword full-text search (tsvector / BM25). Returns raw matching pages. | **No** | Fast lookups, no-spend, exact terms |
| `gbrain query <q>` (alias `ask`) | **Hybrid**: vector + keyword + RRF fusion + multi-query expansion. Returns ranked, deduped results. | Yes (expansion/embeddings) | Conceptual/semantic questions |

**The hybrid scoring stack** (what `query` fuses):
- **Vector** — HNSW over pgvector; semantic similarity from an embedding provider.
- **Lexical** — BM25 keyword.
- **Graph signals** — boosts hubs, cross-source corroboration, demotes session-crowding.
- **Reranker** — ZeroEntropy `zerank-2` (default) or a local cross-encoder.
- **RRF** — reciprocal-rank fusion across all the above.

**Two boost dials** (powerful, often missed):
- `--salience on|strong` — surface emotionally-weighted / activity-rich pages (for "what's going on with me", "anything notable"). The docs explicitly warn: words like "crazy"/"big" often mean *difficult*, not impressive — salience captures that.
- `--recency on` — per-prefix age decay (`daily/`, `chat/` decay fast; `concepts/`, `writing/` stay evergreen).

**Search modes** (cost/quality tradeoff): `conservative` (fewer vector calls, cheapest — *your current setting*) → `balanced` (default) → `tokenmax` (aggressive rerank, highest quality).

**The emotional/activity layer** (don't sleep on this):
- `gbrain salience [--days N]` — pages ranked by mattering, not keywords.
- `gbrain anomalies [--since D] [--sigma N]` — statistical outliers vs your normal cohort.
- `gbrain transcripts recent` — recent raw transcripts (local-only).

---

## 5. The dream cycle — GBrain's built-in *loop*

`gbrain dream` is an **overnight maintenance loop** (cron-friendly; `autopilot --install` runs it as a daemon). It's the engine that turns a pile of notes into a curated brain. Phases:

1. **Entity sweep** — dedupe person/company pages by name + cross-reference.
2. **Citation fixing** — validate claims against sources, flag orphaned citations.
3. **Salience scoring** — rank pages by importance (frequency, recency, centrality).
4. **Contradiction detection** — sample retrieval pairs, LLM judge surfaces conflicting takes.
5. **Consolidation** — merge related notes, prep next-day summaries.

Runs durably on the **Minions job queue** (`gbrain jobs …`: submit/list/get/cancel/retry, two-phase pending→done). This is literally "loop engineering" inside GBrain — and it's the part that **spends LLM tokens unattended**, which is exactly why your install keeps it deferred until you choose to enable it.

> This is the model for the **memory-consolidation loop** in `../loops/loop-plan.md`: dream is GBrain's version of it. Our loop will *feed* well-formed pages in so dream has good material to work on.

---

## 6. The skill system + the 89 MCP tools

- GBrain ships **43 curated skills** as tool-agnostic markdown, bundled as a skillpack, routed by `skills/RESOLVER.md`. Categories: signal capture, ingest, enrichment, querying, brain-ops, citation fixing, daily tasks, cron, reports, voice, soul audit, skill creation, eval, migrations.
- On your install these surface as **89 MCP tools** that any connected agent can call. The three to adopt first (per the install protocol): **signal-detector** (capture on every message), **brain-ops** (brain-first lookup *before* hitting external APIs), **conventions** (citation/backlink/attribution rules).
- Skills are *trainable*: `gbrain skillopt` treats them as parameters — generate benchmarks, propose edits, keep only measurably-better ones. (`check-resolvable` validates the skill tree for reachability / MECE / no-dupes.)

---

## 7. Full-potential capabilities (the stuff beyond "notes + search")

Your install has a lot the README under-sells:

- **Ideation:** `gbrain brainstorm <q>` (bisociation: hybrid search + a "far set" + a judge) and `gbrain lsd <q>` ("Lateral Synaptic Drift" — inverted-judge, rewards far-from-obvious + axiomatic inversions). Creativity tools, not just recall.
- **Code intelligence ("Cathedral II"):** index a codebase as pages and run `code-def`, `code-refs`, `code-callers`, `code-callees`, plus `query --lang/--symbol-kind`, `--near-symbol`, structural `--walk-depth`. Graph-based code understanding, not text grep. (`sync --strategy code`, `reindex-code`.)
- **Multi-source federation:** `gbrain sources add <id> --path <p>` registers multiple brains/repos; `sync --all`/`--source`. This is the **cross-machine hive-mind path** (each device/agent a source).
- **Agent-authored schema:** `gbrain schema` verbs let agents evolve page types — your brain's shape isn't fixed.
- **Sharing & hygiene:** `publish <page.md> [--password]` (AES-256 shareable HTML, strips private data); `lint` (catch LLM artifacts/placeholder dates); `orphans` (pages with no inbound links); `check-backlinks`.
- **Integrations:** `gbrain integrations` manages "senses + reflexes" — webhooks (email/calendar), voice (Twilio + OpenAI Realtime), mobile capture.

---

## 8. Why it's different from plain RAG or Obsidian

| | Plain RAG | Obsidian | **GBrain** |
|---|---|---|---|
| Source of truth | vectors in a DB | markdown vault | **markdown (git) — vectors derived** |
| Relationships | none | manual `[[links]]` | **auto typed edges + multi-hop traversal** |
| Answers | ranked chunks | backlinks | **synthesized, cited, with gap analysis** |
| Maintenance | none | manual | **overnight dream cycle (dedupe, citations, contradictions)** |
| Multi-agent | — | single vault | **MCP tools + multi-source federation + OAuth scoping** |
| "What matters" | — | — | **salience / anomalies (emotional + activity signal)** |

> The thesis: RAG retrieves; GBrain *reads and reasons*. Graph traversal + synthesis + gap analysis ("what does the brain NOT know") is the leap.

---

## 9. Your install today — what's live vs what's gated

Honest map of what you can actually use *right now* (no embeddings, no LLM spend) vs what unlocks:

| Capability | Status on your box | Unlocks with |
|---|---|---|
| Capture / pages / import / sync | ✅ live | — |
| Auto-link + graph traversal | ✅ live (pattern-based, free) | — |
| `gbrain search` (keyword) | ✅ live | — |
| Salience / anomalies *scores* | ⚠️ partial | dream cycle running |
| `gbrain query` hybrid (vector) | ❌ off | an **embeddings key** (ZeroEntropy `ze-…` or OpenAI) → `embed --stale` |
| Reranker (`zerank-2`) | ❌ off | ZeroEntropy key (or local cross-encoder) |
| Dream cycle (dedupe/citations/contradictions) | ❌ deferred | enable cron + accept DeepSeek token spend |
| `brainstorm` / `lsd` / synthesis | ❌ off | LLM enabled (DeepSeek wired, currently dormant) |

**Bottom line for "using it correctly":** today GBrain is a graph + keyword store. Its *brain* (semantic recall, synthesis, overnight curation) is gated behind (a) an embeddings provider and (b) turning the dream cycle on. Both are deliberate, cost-controlled choices — not bugs.

---

## 10. Real use cases

**For you / HiveMind specifically:**
1. **Cross-agent shared memory** — the original goal. Capture from Hermes, Claude Code, Codex into one brain over MCP; "search before acting" becomes every agent's first move. Multi-source federation = the cross-machine version.
2. **Curated fact layer** — relationships and durable facts (who/what/how-connected), *distinct* from preferences (Honcho/`USER.md`) and raw logs (sessions/CASS). Keep the seam clean.
3. **Code intelligence** — index your active repos (`ia-bridge-mcp`, etc.); ask "who calls this?", "blast radius of changing X?" with graph accuracy.

**General high-leverage patterns:**
4. **Meeting/idea capture** → voice/webhook into inbox → dream consolidates overnight → searchable, linked, deduped.
5. **Research brain** — dump articles/PDFs, let auto-link + dream build the concept graph, then `brainstorm`/`lsd` for non-obvious connections.
6. **"What's going on with me"** — `salience`/`anomalies` over daily notes surfaces what actually matters or is anomalous, without you remembering keywords.
7. **Personal CRM** — people/company pages with typed edges (`works_at`, `invested_in`) you can traverse.

---

## 11. How to get full potential (practical roadmap)

1. **Capture relentlessly, structure lightly.** The graph is only as good as its input. Use path prefixes (`people/`, `concepts/`, `daily/`) so type inference and recency decay work. Use `[[wikilinks]]` so auto-link wires edges for free.
2. **Fix the empty-graph problem.** Your current brain is 1 blob page, 0 links — because raw `MEMORY.md` was dumped in. GBrain needs *discrete, well-formed pages* to build a graph. This is exactly what the **memory-consolidation loop** (`../loops/loop-plan.md`) is for: it produces clean per-fact pages instead of one blob.
3. **Add an embeddings provider when semantic recall starts to hurt.** Keyword + graph carries you far; add ZeroEntropy (`ze-…`, GBrain's default reranker+embedder) when you notice "I know it's in there but search can't find it."
4. **Turn on the dream cycle deliberately** — start `--dry-run`, read what it proposes (dedupes, contradictions), then let it run on a schedule. Remember: cron doesn't inherit `GBRAIN_HOME` — set it in the job.
5. **Adopt the 3 core skills** (signal-detector, brain-ops, conventions) so agents capture and look-up consistently.
6. **Stay the engineer.** Read what dream writes; keep the graph *curated, not exhaustive*. A brain that remembers everything is noise; the value is in what it chooses to keep and connect.

---

## 12. Pitfalls (using it correctly)

- **Garbage in → empty/garbage graph.** Dumping unstructured blobs gives GBrain nothing to link. Structure + curation is the work.
- **Unattended dream = unattended spend + unattended mistakes.** Gate it; review its output.
- **Two sources of truth.** Don't let GBrain hoard *preferences* — those belong in Honcho/`USER.md`. GBrain = relationships + facts. Keep the seam.
- **Embeddings/chat provider coupling.** (See install guide §4.) DeepSeek does chat, not embeddings — don't let one clobber the other.
- **"It remembers, so I'll stop thinking."** The brain amplifies judgment; it doesn't replace it.

---

### Quick command cheat-sheet (your install)

```bash
# all commands assume: export GBRAIN_HOME=/home/agent/gbrain-data
gbrain capture "a thought"              # get something in
gbrain import ~/notes/ --no-embed       # bulk in, no spend
gbrain search "term"                    # keyword (free)
gbrain query "conceptual question"      # hybrid (needs embeddings)
gbrain graph <slug> --depth 2           # walk the graph
gbrain backlinks <slug>                 # what links here
gbrain salience --days 7                # what mattered lately
gbrain stats                            # brain overview
gbrain dream --dry-run                  # preview the maintenance loop
gbrain doctor --json                    # health check
gbrain list --type person               # browse by type
```
