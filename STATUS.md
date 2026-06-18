# Status & Roadmap

The living state of the project. The [README](README.md) stays stable and conceptual; this file is where progress is tracked and updated.

**Last updated:** 2026-06-18.

---

## Now / Next / Later

- **Now:** First **manual memory-consolidation cycle** done and verified (issue #1). The brain went from 1 blob page / 0 links to **3 typed pages / 4 links**, all no-spend. The write-policy is captured. See [`gbrain/first-cycle-explained.md`](gbrain/first-cycle-explained.md).
- **Next:** encode the [write-policy](loops/consolidation-write-policy.md) as a Hermes skill and flip the `memory_monitor` cron to **propose-only** (loop Phase A — proposes, operator approves). See [`loops/loop-plan.md`](loops/loop-plan.md).
- **Later:** checker-gated auto-write (Phase B); wire a second agent (Claude Code) to the same brain; GBrain Phase 2 (embeddings + dream cycle); cross-machine federation.

---

## Current state (verified 2026-06-10)

**Hermes** runs on host `mona`, with its own markdown memory (`MEMORY.md`/`USER.md`), an idle cron heartbeat, and an FTS recall DB — i.e. the loop surface already exists.

**GBrain Phase 1 — installed, isolated, and independently verified:**
- `gbrain v0.42.38.0`, PGLite brain, wired into a **dedicated `gbrain` Hermes profile** over MCP — the default Hermes personality is untouched.
- Cost-controlled / no-spend: `conservative` mode, embeddings **off**, dream cycle **deferred**.
- A security issue (an API key hard-coded during install) was caught, the key **rotated**, and the config converted to a single-source env **reference** — verified resolving correctly.
- Robustness fixes applied: `GBRAIN_HOME` persisted; the empty default-brain trap removed.

**Memory-consolidation cycle (2026-06-18) — first real value:**
- The dirty `MEMORY.md` blob (8 raw facts) was curated into **3 clean, typed, wikilinked pages** in a private git-versioned brain repo (`~/brain` on `mona`, never pushed), then projected into GBrain.
- Stale facts dropped (`kate` identity, `ia-bridge-mcp`, `Kimi`), preferences routed to `USER.md`, ephemeral noise dropped. Knowledge graph now **3 pages / 4 links**; lexical search + graph traversal verified returning the right pages.
- Lesson banked: GBrain only builds edges from **full-slug** wikilinks (`[[tools/hermes]]`), not bare cross-directory ones — now a rule in the write-policy.
- Done **no-spend** (embeddings off, no LLM synthesis, `extract` is pattern-matching); default Hermes profile untouched.

**Honest assessment:**
- Semantic search, synthesis, and the dream cycle remain **gated** behind (a) an embeddings provider key and (b) approving LLM spend — both deliberate.
- Net: the pipeline (curate → project → retrieve) is proven on real data; the write-policy is the reusable artifact. Value now grows via more curated facts → automation → sharing.

---

## Roadmap

| Phase | What | State |
|---|---|---|
| Theory & analysis | Hive Mind problem, layered stack, critique | ✅ done |
| GBrain Phase 1 | install, isolate, secure, verify | ✅ done |
| Revoke old key | operator action on the DeepSeek dashboard | ⏳ pending (operator) |
| Manual consolidation cycle | curate `MEMORY.md` → clean pages → graph → retrieval (issue #1) | ✅ done |
| **Consolidation loop (Phase A)** | encode write-policy as a skill; flip `memory_monitor` to propose-only | ▶️ next |
| Loop Phase B | checker-gated auto-write | later |
| Second agent on the brain | wire Claude Code to the same graph (remote MCP) | later |
| GBrain Phase 2 | embeddings key → semantic search; enable the dream cycle | later |
| Cross-machine | multi-source federation (real Postgres) for a true shared brain | later |

---

## Log

- **2026-06-18** — Added **BitmapForge** (`projects/bitmapforge`) as the first tracked HiveMind project, linked to the `hermes` hub. Brain now **4 pages / 6 links**. (The project's *code* stays at `~/Projects/BitmapForge`; the brain holds only a one-page fact about it.)
- **2026-06-18** — First manual memory-consolidation cycle (issue #1): `MEMORY.md` blob → 3 typed/wikilinked pages in a private `~/brain` git repo → projected to GBrain (3 pages / 4 links), lexical search + traversal verified, no-spend. Stale facts dropped, preferences routed to `USER.md`. Wrote [`loops/consolidation-write-policy.md`](loops/consolidation-write-policy.md) and [`gbrain/first-cycle-explained.md`](gbrain/first-cycle-explained.md).
- **2026-06-10** — Reorganized docs into `gbrain/`, `loops/`, `archive/`; README slimmed to stable overview; status split out here.
- **2026-06-10** — GBrain Phase 1 finalized and verified (GBRAIN_HOME persisted, brain resolves from a fresh shell, empty `~/.gbrain` removed, old key absent everywhere → safe to revoke).
- **2026-06-10** — Security: hard-coded DeepSeek key caught during install; rotated and converted to an env reference.
- **2026-06-10** — GBrain installed on `mona` (Phase 1: isolated profile, no-spend, conservative mode).
