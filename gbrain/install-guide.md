# GBrain Installation Guide — for the Hermes agent on `mona`

**Audience:** the Hermes agent running on host `mona` (you, the agent reading this).
**Author:** Claude Code (orchestrating the HiveMind project), 2026-06-10.
**Goal:** install GBrain as an isolated, low-cost relationships/knowledge-graph layer that any agent can reach over MCP — **without disturbing the default `hermes` personality, its memory, or its sessions.**

> Hermes agent: follow THIS guide, not GBrain's `autopilot --install` or auto-remediation. This guide adapts the official `INSTALL_FOR_AGENTS.md` for our specific constraints (DeepSeek-only keys, isolated profile, cost discipline). After each major section, **report back to the operator with the command output before proceeding** — the operator's standing policy (`~/.hermes/memories/USER.md`) requires audit + explicit approval before changes that affect the default profile or spend money.

---

## 0. Decisions already made (do not re-litigate)

1. **Isolation, not reinstall.** The existing Hermes install stays. GBrain is a *separate service*; we wire it into a **cloned, dedicated Hermes profile** so the default personality/memory/sessions are untouched. The "messy" default `MEMORY.md` is kept on purpose — it is test data for a later memory-consolidation loop.
2. **Storage = PGLite** (default). Single machine, no server, good to ~50K pages. (Native Postgres 17 exists on `mona`; only switch to it later if the brain must be shared across machines.)
3. **Phase 1 retrieval = raw lexical/graph search, not synthesis.** Do not spend chat tokens during the initial smoke test. Use `gbrain search` first; only run `gbrain query`/`gbrain think` after the operator approves LLM-backed synthesis.
4. **Chat/LLM provider = DeepSeek, but only if GBrain can separate chat from embeddings.** DeepSeek is OpenAI-compatible for chat, reusing `DEEPSEEK_API_KEY` already in `~/.hermes/.env`, but it does **not** provide embeddings. Do not blindly set `OPENAI_API_KEY=DEEPSEEK_API_KEY` unless the installed GBrain version exposes distinct chat/provider/base-URL knobs that will not route embedding calls to DeepSeek.
5. **Embeddings = DISABLED to start** (lexical + graph retrieval only). Reason: `mona` has no embeddings-capable key (DeepSeek/Kimi/OpenRouter have no embeddings endpoint; GBrain's default ZeroEntropy needs a `ze-...` key we don't have). Semantic vector search is the documented upgrade in §7 once an embeddings key exists.
6. **Cost mode = `conservative`.** No `--auto --max-usd`, no `autopilot`, no paid embeddings, no LLM synthesis until the operator approves.

---

## 1. Install Bun + GBrain

```bash
# Bun (GBrain's runtime)
curl -fsSL https://bun.sh/install | bash
export PATH="$HOME/.bun/bin:$PATH"     # add to ~/.zshrc too so cron/MCP find it

# GBrain
bun install -g github:garrytan/gbrain
gbrain --version
which gbrain
```

If the global install fails, use the deterministic path:
```bash
git clone https://github.com/garrytan/gbrain.git ~/gbrain && cd ~/gbrain
bun install && bun link
gbrain --version
which gbrain
```

If `bun install -g` aborts or `gbrain doctor` later reports `schema_version: 0`, run the upstream recovery command before continuing:
```bash
gbrain apply-migrations --yes
```

**→ Report:** `gbrain --version` output and the resolved `gbrain` path (`which gbrain`). The path matters — cron and MCP must use an absolute path.

---

## 2. Discover the real config surface (do this before trusting any flag below)

GBrain's exact config keys vary by version. **Enumerate them on the installed build instead of guessing:**

```bash
gbrain --help
gbrain config --help
gbrain config list        # or: gbrain config get   (use whichever exists)
gbrain init --help
gbrain doctor --help
gbrain models --help        # if present
gbrain search modes         # if present
```

Specifically find the keys/vars that control: **(a)** the chat model + provider, **(b)** an OpenAI-compatible **base-URL override**, **(c)** how to **disable embeddings** / choose a search mode that doesn't require them, **(d)** the PGLite DB path, **(e)** whether chat and embeddings can use separate providers.

**→ Report:** the relevant lines from `--help`/`config list` so we confirm the exact knob names before writing config. If a name below differs on this version, use the real one and note the substitution.

---

## 3. Create the brain (PGLite, isolated location)

Keep the brain out of `~/.hermes` so it is clearly a separate layer:

```bash
mkdir -p ~/gbrain-data && cd ~/gbrain-data
gbrain init --pglite        # explicit PGLite; no server
gbrain doctor --json        # expect DB checks to pass; embeddings/keys will FLAG — that is expected for now
gbrain config list          # verify engine + data path; use the installed build's real command if this differs
```

**Important isolation check:** upstream GBrain has documented PGLite storage at `~/.gbrain/brain.db` by default unless configured. Do **not** assume `cd ~/gbrain-data` makes the DB live there. Confirm the actual PGLite path from config/doctor output. If the DB path is outside `~/gbrain-data`, either explicitly configure it to an isolated path (using the installed version's real key) or report the path and pause for operator approval before continuing.

**→ Report:** full `gbrain doctor --json`, config output showing engine + DB path, and `gbrain search modes` if available. We expect green on DB/runtime and red/warn on embeddings + missing keys — that is the intended starting state, not a failure.

---

## 4. Configure no-spend retrieval first; add DeepSeek chat only if safe

The initial target is **no-spend raw retrieval**. Configure conservative search mode and keep embeddings off. Do **not** set any API key until the installed build proves that chat and embedding providers can be separated.

```bash
gbrain config set search.mode conservative
gbrain search modes         # verify conservative is active, if this command exists
gbrain models               # inspect configured model touchpoints, if this command exists
gbrain models doctor --json # expect embedding/chat warnings with no keys; this is acceptable for Phase 1
```

Only if §2 found distinct chat-provider/base-URL/model knobs that do **not** affect embeddings, configure DeepSeek for chat. **Do NOT put the DeepSeek key in the default Hermes env** — scope it to GBrain.

Create `~/gbrain-data/.env` only if §3 confirmed that `~/gbrain-data` is the correct working/config directory for this install:

```bash
# Reuse the DeepSeek key Hermes already has, without copying the secret by hand:
DS_KEY=$(grep -E '^DEEPSEEK_API_KEY=' ~/.hermes/.env | cut -d= -f2-)
test -n "$DS_KEY" || { echo "DEEPSEEK_API_KEY missing from ~/.hermes/.env"; exit 1; }

cat > ~/gbrain-data/.env <<EOF
# Use the installed GBrain build's real chat-only env/config names here.
# Do not use OPENAI_API_KEY / OPENAI_BASE_URL for DeepSeek unless the build
# proves those variables are chat-only and cannot trigger embedding calls.
EOF
chmod 600 ~/gbrain-data/.env
```

Then set the chat model to a DeepSeek model via the **real** chat-only config key you found in §2. Examples below are placeholders; do not run them unless they exist on this build and are chat-only:

```bash
# placeholder shape only — use real chat-only keys from §2:
gbrain config set chat.model deepseek-chat
gbrain config set chat.base_url https://api.deepseek.com/v1
```

**IMPORTANT — embeddings caveat:** If the only available path is `OPENAI_API_KEY` + `OPENAI_BASE_URL`, skip DeepSeek configuration in Phase 1. Because DeepSeek serves no embeddings, using it as generic OpenAI config can break embedding probes/vector calls. Do not run `gbrain embed` until §7.

**→ Report:** the final `gbrain config list` and `gbrain models doctor --json` (redact keys) so the operator can confirm mode=conservative, embeddings not active, and either chat=unconfigured for no-spend Phase 1 or chat=DeepSeek via chat-only knobs.

---

## 5. Import seed data WITHOUT embeddings, verify graph + lexical search

Start with a tiny, safe corpus to prove the pipeline. **Do not import secrets or the whole home dir.** Also do not import preferences into GBrain in Phase 1: per `../loops/loop-plan.md`, preferences belong in `USER.md`/Honcho, while GBrain should own relationships and curated factual recall.

Good first corpus: a sanitized copy of `MEMORY.md` only. Review it before import and remove secrets, ephemeral approvals, and preference-like facts.

**Before running `extract`, confirm it is LLM-free.** GBrain's auto-link on page write is documented as "pure pattern matching, no LLM calls," but `gbrain extract links/timeline --source db` may be richer and call the chat model — which would break the no-spend Phase 1. Verify from §2's help output (or run only the `--dry-run` and inspect whether it makes network/LLM calls). **If `extract` is LLM-backed, do NOT run the non-dry-run forms in Phase 1 — defer graph extraction to Phase 2** (after the operator approves chat spend) and proceed with import + lexical search only.

```bash
mkdir -p ~/gbrain-data/seed
cp ~/.hermes/memories/MEMORY.md ~/gbrain-data/seed/   # known-messy on purpose
# Do not copy USER.md into GBrain during Phase 1.

gbrain import ~/gbrain-data/seed/ --no-embed          # NO embeddings
gbrain extract links --source db --dry-run | head -20 # preview only; safe even to inspect first
# Run the next two ONLY if §2 confirmed extract is pattern-based / LLM-free:
gbrain extract links --source db                      # build the knowledge graph
gbrain extract timeline --source db
gbrain stats
# Raw, no-LLM retrieval. `gbrain search` is the EXPECTED command name — confirm it exists
# and is non-synthesizing in §2; if not, use the real raw-retrieval command (NOT query/think):
gbrain search "Kimi context-window bug"
```

A useful signal: GBrain's contradiction/dedup machinery should eventually notice the **two near-duplicate "Kimi context-window bug" entries** in `MEMORY.md`. Note whether raw search / `gbrain stats` surfaces them.

**→ Report:** whether `extract` was confirmed LLM-free (and whether it ran or was deferred), `gbrain stats`, and the raw-search output. If lexical+graph search returns sensible results, the core layer works without paying for embeddings or chat synthesis.

---

## 6. Wire GBrain into an ISOLATED Hermes profile over MCP

Do **not** add the GBrain MCP server to the default `hermes` profile. First discover the real Hermes profile/MCP surface without mutating anything:

```bash
hermes --help
hermes profile --help
hermes mcp --help        # or the closest equivalent in this Hermes version
```

**→ Report:** the relevant Hermes help output and the exact command/config path you propose to modify. Pause for operator approval before creating the profile.

After approval, clone a dedicated profile:

```bash
hermes profile create gbrain --clone      # isolated personality/session state
hermes profile alias gbrain               # per operator preference; avoid `profile use` (sticky default)
```

Register GBrain as an MCP server **for the `gbrain` profile only**. Use the absolute `gbrain` path from §1 and the verified GBrain working/config directory from §3:

```bash
# Expected shape only (adapt to the real CLI/config):
#   command: /abs/path/to/gbrain
#   args:    ["serve"]
#   cwd:     /home/agent/gbrain-data  # only if §3 confirmed this is the right runtime/config dir
```

If Hermes registers MCP servers via `config.yaml`, edit **only the `gbrain` profile's** config (under `~/.hermes/profiles/gbrain/`), never the root `~/.hermes/config.yaml`. Show the diff before saving.

**→ Report:** the exact MCP registration you made and the profile path it landed in. **Pause here for operator approval before first launch.**

---

## 7. (LATER — needs operator approval) Enable semantic embeddings + nightly dream cycle

Only after §1–6 are confirmed working and the operator approves spend:

- **Embeddings:** obtain a `ZEROENTROPY_API_KEY` (`ze-...`, GBrain's default embedder/reranker) OR a real embeddings-capable `OPENAI_API_KEY`. Put it in the verified GBrain env/config location **without** a DeepSeek base-URL override clobbering embedding calls (embeddings and chat may need separate provider config — verify in §2's config surface). Then:
  ```bash
  gbrain embed --stale
  gbrain doctor --json     # embeddings should now pass
  ```
- **Recurring jobs (the loop / "dream cycle"):** register cron using **absolute paths**, with the working directory and markdown repo path verified explicitly:
  ```bash
  # every 15 min:  gbrain sync --repo /path/to/curated-markdown-truth && gbrain embed --stale
  # nightly:       gbrain dream        # 8-phase: entity sweep, citation fixes, memory consolidation, contradictions
  # weekly:        gbrain doctor --json
  ```
  `sync --repo` must point at the curated markdown source of truth, not the GBrain runtime/data directory unless those are intentionally the same and verified.
  Prefer explicit cron over `gbrain autopilot --install` so the schedule is auditable. **Do not enable nightly `gbrain dream` until the operator confirms** — it is the part that spends DeepSeek tokens unattended (this is exactly the "loop running unattended is also a loop making mistakes unattended" risk).

---

## 8. Guardrails (standing rules for this install)

- Touch the **default `hermes` profile / root config: never.** All GBrain wiring lives in the `gbrain` profile plus GBrain's own data/config dir — which is the path **verified in §3** (may be `~/gbrain-data`, may be GBrain's default `~/.gbrain/`), not assumed.
- **No unattended spend:** no `--auto`, no `--max-usd`, no `autopilot`, no nightly `dream`, no paid embeddings, no LLM synthesis (`query`/`think`) — until the operator explicitly approves each.
- **Secrets:** reuse `DEEPSEEK_API_KEY` by reference (§4); never echo key *values* into reports or logs.
- **Report-and-pause** at the end of §1, §3, §4, §5, and §6. Show command output / diffs; wait for approval before steps that affect profiles, config, or money.
- If any flag/key name in this guide doesn't exist on the installed version, use the real one from §2 and **note the substitution** in your report.

---

## 9. Definition of done (Phase 1)

- `gbrain --version` works; PGLite initialized; actual DB/config path verified and approved.
- `conservative` mode active; embeddings off; chat either unconfigured for no-spend Phase 1 or configured to DeepSeek only through chat-only knobs.
- Seed corpus imported `--no-embed`; raw lexical search returns sensible answers without paid embeddings or LLM synthesis. (Graph extraction either ran — if confirmed LLM-free — or is explicitly deferred to Phase 2.)
- A `gbrain` Hermes profile exists with the GBrain MCP server registered (default profile untouched).
- Operator has all reports and has approved before any profile mutation, LLM synthesis, nightly, or paid step.

> This GBrain instance is the **"relationships" layer** of the HiveMind stack (see `../loops/loop-plan.md` §3). The next milestone is the **memory-consolidation loop**: a scheduled maker/checker loop that cleans `MEMORY.md` and projects curated facts into this graph — at which point the dirty seed data from §5 becomes the first real job.
