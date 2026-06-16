# Status & Roadmap

The living state of the project. The [README](README.md) stays stable and conceptual; this file is where progress is tracked and updated.

**Last updated:** 2026-06-10.

---

## Now / Next / Later

- **Now:** GBrain Phase 1 is complete and verified. Revoking the old DeepSeek API key is the only open operator action.
- **Next:** build the first loop — the **memory-consolidation loop** (see [`loops/loop-plan.md`](loops/loop-plan.md)). This also fixes the empty-graph problem below.
- **Later:** GBrain Phase 2 (embeddings + dream cycle), then cross-machine federation.

---

## Current state (verified 2026-06-10)

**Hermes** runs on host `mona`, with its own markdown memory (`MEMORY.md`/`USER.md`), an idle cron heartbeat, and an FTS recall DB — i.e. the loop surface already exists.

**GBrain Phase 1 — installed, isolated, and independently verified:**
- `gbrain v0.42.38.0`, PGLite brain, wired into a **dedicated `gbrain` Hermes profile** over MCP — the default Hermes personality is untouched.
- Cost-controlled / no-spend: `conservative` mode, embeddings **off**, dream cycle **deferred**.
- A security issue (an API key hard-coded during install) was caught, the key **rotated**, and the config converted to a single-source env **reference** — verified resolving correctly.
- Robustness fixes applied: `GBRAIN_HOME` persisted; the empty default-brain trap removed.

**Honest assessment:**
- The knowledge graph is currently **one page, zero links** — `MEMORY.md` was imported as a single blob. Not a bug; it's the central lesson: **GBrain needs discrete, well-formed pages to build relationships.**
- Semantic search, synthesis, and the dream cycle are **gated** behind (a) an embeddings provider key and (b) approving LLM spend — both deliberate.
- Net: the *plumbing* is correct and safe; the *value* is unlocked by structured input + curation, not more configuration.

---

## Roadmap

| Phase | What | State |
|---|---|---|
| Theory & analysis | Hive Mind problem, layered stack, critique | ✅ done |
| GBrain Phase 1 | install, isolate, secure, verify | ✅ done |
| Revoke old key | operator action on the DeepSeek dashboard | ⏳ pending (operator) |
| **Memory-consolidation loop** | first real loop; produces clean pages, fixes the empty-graph problem | ▶️ next |
| GBrain Phase 2 | embeddings key → semantic search; enable the dream cycle | later |
| Cross-machine | multi-source federation (real Postgres) for a true shared brain | later |

---

## Log

- **2026-06-10** — Reorganized docs into `gbrain/`, `loops/`, `archive/`; README slimmed to stable overview; status split out here.
- **2026-06-10** — GBrain Phase 1 finalized and verified (GBRAIN_HOME persisted, brain resolves from a fresh shell, empty `~/.gbrain` removed, old key absent everywhere → safe to revoke).
- **2026-06-10** — Security: hard-coded DeepSeek key caught during install; rotated and converted to an env reference.
- **2026-06-10** — GBrain installed on `mona` (Phase 1: isolated profile, no-spend, conservative mode).
