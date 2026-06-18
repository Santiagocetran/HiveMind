# Memory-Consolidation Write-Policy

**What this is:** the rules for turning raw agent memory into clean, durable brain pages — *what* deserves to be remembered, *where* each fact belongs, and *how* a page is shaped. It is the reusable artifact produced by the first manual consolidation cycle (issue #1), and the future loop's **skill**: when the `memory_monitor` loop is turned on, its checker enforces exactly these rules.

**Status:** validated by one manual cycle on `mona` (2026-06-18). The brain went from 1 blob page / 0 links to **3 typed pages / 4 links**, all no-spend. See [`../gbrain/first-cycle-explained.md`](../gbrain/first-cycle-explained.md) for the narrative + the resulting graph.

**Anchored on:** "Files = truth" ([`../hivemind-analysis.md`](../hivemind-analysis.md)) and the layer seams in [`loop-plan.md`](loop-plan.md) §3.

---

## 0. The one rule everything else serves

> **A fact written to the wrong layer is a bug. A stale or ephemeral fact written at all is noise. The brain is only worth trusting if it stays signal.**

Curation is the hard problem, not storage. This doc is the curation policy.

---

## 1. The seam rule — which layer owns a fact

For every candidate fact, decide its layer *first*. Most "should I remember this?" mistakes are really "this went to the wrong layer" mistakes.

| If the fact is… | It belongs in… | Not in GBrain because… |
|---|---|---|
| A **durable, current fact** (an entity, a project, a tool, an incident, a relationship) | **GBrain** (a curated page) | — this is what GBrain is for |
| A **preference / behavioral policy** ("user prefers X", "always do Y", "act as owner") | **`USER.md`** / Honcho | preferences drive behavior every turn; they're not graph nodes |
| **Ephemeral state** (a one-time approval, "user said yes to the diff", transient status) | **nowhere** — drop it | it was true for one moment; as a durable fact it's a lie tomorrow |
| **Raw history** (full session transcripts, logs) | `sessions/` / CASS (deferred) | the brain holds curated facts, not raw replay |

---

## 2. The classification procedure (run on every candidate fact)

```
For each candidate fact:
  1. Is it a PREFERENCE / behavioral policy?  → route to USER.md, not GBrain.
  2. Is it EPHEMERAL (true only at one moment)? → DROP.
  3. Is it STALE (describes a PAST setup that's no longer real)? → DROP / do not carry forward.
  4. Is it a DUPLICATE / near-duplicate of an existing fact? → MERGE into one page.
  5. Otherwise it's a durable, current fact → write it as a typed page (§4).
```

**The stale-detection step (3) is the one most easily missed.** Raw memory accumulates facts about a *past* reality. The first cycle's seed described an identity (`kate`), a project (`ia-bridge-mcp`), and a model (`Kimi`) that were all retired — projecting them as current would have actively misled future agents. **Verify each fact against current ground truth before writing; drop what's obsolete.** Don't carry a past fact forward "just in case" — git history of the brain repo already preserves what was removed.

**Worked example — the first cycle's seed (`MEMORY.md`, 8 facts):**

| Raw fact | Verdict |
|---|---|
| "This machine is MINE, NOPASSWD sudo, never ask permission" | preference/policy → `USER.md` (scoped: free machine ops; pause only for spend + shared-brain writes) |
| "User prefers isolated Hermes profiles (`--clone`+`alias`)" | preference → `USER.md` |
| "Git config = `kate <kate@lavanguardia.local>`" | **stale** (now `agent <agent@mona.local>`) → replaced with current fact |
| "Active project: ia-bridge-mcp Hermes support" | **stale** (retired) → dropped |
| "ia-bridge-mcp is not a proxy / don't assume architecture" | **stale fact + preference** → dropped + behavioral half to `USER.md` |
| Kimi context-window bug (entry 1) | **stale** (Kimi no longer used) → dropped |
| Kimi context-window bug (entry 2) | duplicate of above + **stale** → dropped |
| "User approved file modifications after I showed the diff" | **ephemeral** → dropped |

Result: the dirty 8-fact blob became **3 clean, current pages** — `tools/hermes`, `tools/gbrain`, `concepts/mona-environment` — plus updates to `USER.md`.

---

## 3. Taxonomy (keep it small)

Pages live in type-prefixed directories. **No complex ontology** — only these, and add a new one only when a fact genuinely doesn't fit:

```
projects/    ongoing work, goals
concepts/    durable ideas, environments, how-things-are
tools/       systems/programs and what they are/do
incidents/   bugs, outages, post-mortems (a thing that happened, with a cause)
people/      durable person-facts only
```

The directory sets the page `type` (GBrain infers it from the path prefix).

---

## 4. Page format

```markdown
---
title: <Human Title>
type: <note|tool|project|incident|person>   # usually matches the directory
tags: [<short>, <lowercase>, <tags>]
---
<One or a few tight sentences of durable, current fact.>
Link related pages with full-slug wikilinks (see §5).
```

Rules:
- **One discrete fact-cluster per page.** If a page is really two unrelated facts, split it.
- **Tight prose, no fluff.** A page is a fact, not an essay.
- **Current tense, present truth.** If it changes, edit the page (the git diff is the history).

---

## 5. Wikilinks — the cross-directory gotcha (learned the hard way)

GBrain builds the graph from `[[...]]` wikilinks via `extract links --source fs` — **pure pattern matching, no LLM, no spend.** But:

> **`extract --source fs` resolves a bare `[[name]]` relative to the source file's own directory.** A link that crosses directories with a bare name **silently fails** — it produces no edge and no error.

So `[[hermes]]` inside `tools/gbrain.md` resolves (same folder), but `[[hermes]]` inside `concepts/mona-environment.md` does **not** (it looks for `concepts/hermes`). This is exactly what kept the first cycle's graph stuck at 2 links instead of 4.

**The rule: always use the full slug in wikilinks.**

```markdown
[[tools/hermes]]              ✅ resolves from anywhere
[[concepts/mona-environment]] ✅ resolves from anywhere
[[hermes]]                    ⚠️ only resolves within tools/
```

Also: **don't glue punctuation onto a wikilink** — `[[tools/hermes]]'s machine` can break the parser. Write `[[tools/hermes]] owns this machine`.

---

## 6. Projection rules — truth → index

- **Truth is a versioned, git-diffable markdown brain repo** (on `mona`: `~/brain`, private, never pushed to the public HiveMind repo). GBrain/PGLite is a **rebuildable projection**, never the sole home of a fact.
- Project with **`gbrain sync --repo <brain> --no-embed`** (Phase 1). `sync` tries to embed by default and will *refuse* (no embeddings key) rather than spend — so `--no-embed` is mandatory, not optional.
- Then **`gbrain extract links --source fs --dir <brain>`** to build/refresh the graph (idempotent, LLM-free).
- Keep `README`/scaffolding out of the import — name it `README.txt` (not `.md`) so `sync` doesn't ingest it as an orphan page.
- `gbrain delete <slug>` is a **soft-delete** with a 72h recovery window (it auto-hard-purges after). Deleted content disappears from search/list/get immediately; the tombstone chunk lingers in `stats` counts until purge — that's expected, not a leak.

---

## 7. No-spend posture (Phase 1)

Everything in a consolidation cycle must be free:
- Embeddings **off** (`embedding_disabled: true`); always `--no-embed`.
- Retrieval uses **`gbrain search`** (lexical/tsvector) and **graph traversal** — never `query`/`ask` (hybrid, may expand via LLM) and never `dream` (the paid nightly cycle).
- `extract links` is pattern-matching only — confirm with `--dry-run` before the real pass.

Semantic embeddings + the dream cycle are a later, explicitly-approved phase.

---

## 8. Maker / checker discipline (how the write actually happens)

Per the operator policy in `USER.md`, a write to the shared brain is not a solo act:

1. **Maker** drafts the curated pages (the markdown truth) from the raw memory.
2. **Checker** audits them — Codex CLI / a second opinion — against this policy: right layer? stale? duplicate? seam respected?
3. **Operator approves**, then the write lands via the `gbrain` profile.
4. **Verify** the cycle (§9).

When this is encoded as the `memory_monitor` loop, it starts **propose-only** (Phase A): the loop writes proposals, the operator approves, nothing auto-merges. The checker earns auto-write (Phase B) only once it's boring. *Read every fact the loop writes until then.*

---

## 9. Per-cycle verification checklist

A cycle isn't done until:

- [ ] Every candidate fact was classified (§2) — preferences routed to `USER.md`, ephemeral/stale dropped, dupes merged.
- [ ] Pages are typed, single-fact, and use **full-slug wikilinks** (§5).
- [ ] `gbrain stats` shows the expected page count and **links > 0**.
- [ ] `gbrain search "<a known fact>"` returns the right page (lexical, no-spend).
- [ ] `gbrain graph <hub-slug> --depth 2` and `gbrain backlinks <hub-slug>` traverse correctly.
- [ ] No spend: embeddings off, no `query`/`ask`/`dream`, all `--no-embed`.
- [ ] Boundaries intact: only the brain repo + `gbrain` profile/data touched (default Hermes profile changes, like a `USER.md` edit, are a separate, explicitly-approved action).
