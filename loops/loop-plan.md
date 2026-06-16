# The Memory Loop — Loop Engineering applied to Hermes

**Date:** 2026-06-10
**Author:** Claude (grounded in live recon of `mona` @ 10.10.20.37 + Osmani "Loop Engineering" + `hivemind-analysis.md`)
**Status:** Design doc. Decides layer seams *before* code (per `hivemind-analysis.md` §5). No code written yet.

---

## 0. TL;DR

- A **loop** = a closed control cycle (`trigger → discover → execute → verify → act → record → repeat`) where the output of each turn feeds the next *without you in the chair*. You stop being the runtime; you design it.
- Osmani's six primitives: **automations** (heartbeat), **worktrees** (isolation), **skills** (persistent knowledge), **connectors/MCP** (reach into real tools), **sub-agents** (split *maker* from *checker*), **state/memory** (on disk, outside the conversation).
- **The memory layer is the loop's spine, not a separate project.** GBrain's "overnight consolidation" *is itself a loop*. Our HiveMind work and loop engineering are two halves of one machine.
- **Reality on `mona`:** Hermes already has the Files=truth layer (`~/.hermes/memories/*.md`), an idle cron heartbeat (`memory_monitor`, `cron_mode: deny`), a recall DB (`state.db`), a stubbed continuity slot (`honcho: {}`), and **no GBrain**. The curation failure mode (dupes + noise) is *already present* in `MEMORY.md`.
- **First loop to build:** a **memory-consolidation loop** over `MEMORY.md` whose *checker operates on the writes* — because, per our own analysis, **the write/curation policy is the hard problem, not storage.** The checker is the Codex/ia-bridge audit the user already trusts (USER.md).

---

## 1. What a loop is (the concept, briefly)

> "Loop engineering is replacing yourself as the person who prompts the agent. You design the system that does it instead." — Osmani

| Primitive | Role | What happens without it |
|---|---|---|
| Automation | the heartbeat — runs on a schedule, does discovery/triage | nothing starts on its own; you're still the trigger |
| Worktree | isolation — each agent on its own branch | parallel agents stomp each other |
| Skill | persistent project knowledge (`SKILL.md`), reused every run | you re-explain context every session |
| Connector (MCP) | reach into real tools | agent *says* "here's the fix" instead of *doing* it |
| Sub-agent | split **maker** from **checker** | self-grading bias; no one to trust when you walk away |
| State / Memory | what's done + next, **on disk** | the model forgets everything between runs |

Two control modes: **`/loop`** (cadence-based) and **`/goal`** (run until a verifiable condition is true, checked by a small model each turn).

**Pitfalls to design against** (Osmani): comprehension debt, cognitive surrender, *unverified automation* ("a loop running unattended is also a loop making mistakes unattended"), token cost. Plus ours: **uncritical writes turn a memory loop into a noise pump.**

---

## 2. Ground truth on `mona` (live recon 2026-06-10)

```
~/.hermes/
  memories/MEMORY.md     ← Files=truth, append-only, "§"-delimited facts  [EXISTS]
  memories/USER.md       ← preferences / approval policy                  [EXISTS]
  cron/                  ← empty; heartbeat surface                       [IDLE]
  hooks/                 ← empty                                          [IDLE]
  state.db (28MB)        ← session store + FTS5 recall                    [EXISTS]
  skills/ (30 cats)      ← incl. autonomous-ai-agents, note-taking/obsidian
  sessions/, checkpoints/, logs/
config.yaml:
  memory.memory_enabled: true        memory.memory_char_limit: 2200
  honcho: {}                         ← continuity slot STUBBED
  cron.cron_mode: deny               cron.memory_monitor: [cronjob, memory]  ← heartbeat OFF
NO gbrain anywhere on the box.
```

**Two corrections to the Grok plan** (`hivemind-first-plan.md`), consistent with `hivemind-analysis.md`:
1. GBrain is **not installed** — it's a future layer, not the current backend. Hermes runs its *own* markdown memory.
2. The integration mechanism is **MCP** (`mcp_serve.py`, `optional-mcps/`), not "direct integration."

**The curation problem is already live** — `MEMORY.md` contains:
- two near-duplicate "Kimi K2.6 context-window bug" entries (should be *merged*), and
- ephemeral noise ("User approved file modifications after I presented the diff") that should never have been a durable fact.

This is the exact failure `hivemind-analysis.md` §3.2 predicted. The first loop's job is to *fix and prevent* it.

---

## 3. Layer seams — decided (the part to get right first)

Anchored on **Files = truth** (`hivemind-analysis.md` §4). Everything else is a derived/disposable projection.

| Layer | Owner on `mona` | Status | Decision |
|---|---|---|---|
| **Truth** | `~/.hermes/memories/*.md` | ✅ live | The anchor. The loop reads/writes *here*. Never let a derived store become the source of truth. |
| **Continuity** (prefs, user model) | Honcho (`honcho: {}`) | stubbed | Owns *preferences*. Today `USER.md` does this by hand. Keep prefs in USER.md/Honcho — **don't let them leak into MEMORY.md or GBrain.** |
| **Recall** (fast search) | `state.db` FTS5 | ✅ live | Derived index over truth. Rebuildable. "Search before acting" reads here. |
| **Relationships** (graph) | GBrain | ❌ not installed | **Defer.** When added: **PGLite default** (single machine). Real Postgres+pgvector **only** if memory must be shared across `mona` + other machines (the true hive-mind case). Keep it *curated*, fed *from* `MEMORY.md`. |
| **Episodic / journey** | `sessions/` + CASS | partial / not installed | Defer. Only if faithful replay of *how* you got somewhere starts hurting. |
| **Governance / the loop** | `cron/` + `hooks/` | idle, ready | **Where the memory loop lives.** Currently `cron_mode: deny` — correct safety posture to start from. |

**Seam rule (the write policy, one line):** *facts → MEMORY.md; preferences → USER.md/Honcho; relationships → GBrain (later); raw logs → sessions/CASS.* A fact written to the wrong layer is the bug.

---

## 4. The first loop: Memory-Consolidation Loop (baby dream-cycle)

The smallest loop that exercises **all six primitives** and directly advances the hive-mind goal. It is GBrain's "overnight consolidation," started at the Files=truth layer *before* any graph exists.

```
MEMORY LOOP   (heartbeat = the existing cron `memory_monitor`)
 1. discover : read session deltas since last consolidation watermark
 2. extract  : MAKER sub-agent → propose candidate durable facts
 3. verify   : CHECKER sub-agent → reject ephemeral/noise, merge near-dupes,
               flag contradictions vs existing MEMORY.md, enforce the seam rule
 4. act      : write survivors to MEMORY.md (§-delimited) / USER.md (if preference);
               append to a consolidation log
 5. record   : advance watermark so next run starts after it
```

### Why this loop, specifically
- It uses infrastructure **already present and idle** (cron heartbeat + markdown truth + FTS recall). Nothing new to install.
- **The checker is on the *writes*, not on code** — because our own analysis says curation is the hard problem. This is the key design inversion vs. Osmani's code-fix example.
- The checker role maps onto a verifier the user **already trusts**: the *"audit by Codex/ia-bridge-mcp, then explicit approval"* policy in `USER.md`. We're formalizing an existing manual loop, not inventing one.
- It has an immediate, visible payoff: it will **merge the duplicate Kimi entries and evict the ephemeral noise** on its first real run.

### Primitive mapping
| Primitive | In this loop |
|---|---|
| Automation | `cron memory_monitor` (flip from `deny` → reviewed mode) |
| Skill | a `memory-consolidation` SKILL.md = the write policy + seam rule + §-format |
| Connector | MCP — read `state.db`/sessions; (later) write to GBrain |
| Sub-agent: maker | extracts candidate facts from session deltas |
| Sub-agent: checker | ia-bridge-mcp / Codex audit — noise/dupe/contradiction/seam check |
| State | `MEMORY.md` + `USER.md` + a `consolidation-log.md` + watermark |

### Safety posture (start here, earn autonomy)
Mirror Osmani's *triage-inbox + human escalation* and the user's approval gate:
1. **Phase A — propose-only.** Loop writes proposals to `consolidation-proposals.md`; nothing touches `MEMORY.md` until the user approves. (`cron_mode` stays `deny`-equivalent.)
2. **Phase B — checker-gated auto-write.** Once the checker is trusted, survivors auto-merge; rejects + contradictions escalate to the proposals file. Read the diffs (avoid comprehension debt).
3. **Phase C — extend to GBrain.** Only after A/B are boring: install GBrain (PGLite), and have the loop *also* project curated facts into the graph. MEMORY.md stays the source of truth; the graph is rebuildable from it.

---

## 5. Two ways to "try a loop" this week (both real)

- **(i) On `mona`, native:** author the `memory-consolidation` skill + flip the `memory_monitor` cron to Phase-A propose-only. Most authentic — uses Hermes' real primitives, ties into Honcho/state.db.
- **(ii) Locally, in Claude Code:** prototype the same loop with the `/loop` skill over a sample `MEMORY.md` first, to learn maker/checker mechanics with zero risk to `mona`, then port. Lowest blast radius.

Recommended order: **(ii) to learn → (i) to land.**

---

## 6. Open decisions (gate the next step)

1. **Phase A scope:** prototype locally first (ii), or go straight to the `memory_monitor` cron on `mona` (i)?
2. **Checker identity:** ia-bridge-mcp, Codex CLI, or a second Hermes profile as the checker sub-agent? (USER.md implies ia-bridge/Codex.)
3. **Discovery source:** consolidate from `state.db` sessions, or from a simpler hand-fed delta to start?
4. **GBrain timing:** confirmed deferred to Phase C — agreed? (It only earns its place once memory crosses machines.)

---

## 7. The one-line philosophy (Osmani, kept in view)

> "Build the loop. But build it like someone who intends to stay the engineer, not just the person who presses go."

For a *memory* loop that means: **read every fact it writes, until the checker earns the right to write unread.**
