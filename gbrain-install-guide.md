# GBrain Installation Guide — for the Hermes agent on `mona`

**Audience:** the Hermes agent running on host `mona` (you, the agent reading this).
**Author:** Claude Code (orchestrating the HiveMind project), 2026-06-10.
**Goal:** install GBrain as an isolated, low-cost relationships/knowledge-graph layer that any agent can reach over MCP — **without disturbing the default `hermes` personality, its memory, or its sessions.**

> Hermes agent: follow THIS guide, not GBrain's `autopilot --install` or auto-remediation. This guide adapts the official `INSTALL_FOR_AGENTS.md` for our specific constraints (DeepSeek-only keys, isolated profile, cost discipline). After each major section, **report back to the operator with the command output before proceeding** — the operator's standing policy (`~/.hermes/memories/USER.md`) requires audit + explicit approval before changes that affect the default profile or spend money.

---

## 0. Decisions already made (do not re-litigate)

1. **Isolation, not reinstall.** The existing Hermes install stays. GBrain is a *separate service*; we wire it into a **cloned, dedicated Hermes profile** so the default personality/memory/sessions are untouched. The "messy" default `MEMORY.md` is kept on purpose — it is test data for a later memory-consolidation loop.
2. **Storage = PGLite** (default). Single machine, no server, good to ~50K pages. (Native Postgres 17 exists on `mona`; only switch to it later if the brain must be shared across machines.)
3. **Chat/LLM provider = DeepSeek** (OpenAI-compatible), reusing `DEEPSEEK_API_KEY` already in `~/.hermes/.env`. No new chat spend.
4. **Embeddings = DISABLED to start** (lexical + graph retrieval only). Reason: `mona` has no embeddings-capable key (DeepSeek/Kimi/OpenRouter have no embeddings endpoint; GBrain's default ZeroEntropy needs a `ze-...` key we don't have). Semantic vector search is the documented upgrade in §7 once an embeddings key exists.
5. **Cost mode = `conservative`.** No `--auto --max-usd`, no `autopilot`, no paid embeddings until the operator approves.

---

## 1. Install Bun + GBrain

```bash
# Bun (GBrain's runtime)
curl -fsSL https://bun.sh/install | bash
export PATH="$HOME/.bun/bin:$PATH"     # add to ~/.zshrc too so cron/MCP find it

# GBrain
bun install -g github:garrytan/gbrain
gbrain --version
```

If the global install fails, use the deterministic path:
```bash
git clone https://github.com/garrytan/gbrain.git ~/gbrain && cd ~/gbrain
bun install && bun link
gbrain --version
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
```

Specifically find the keys/vars that control: **(a)** the chat model + provider, **(b)** an OpenAI-compatible **base-URL override**, **(c)** how to **disable embeddings** / choose a search mode that doesn't require them.

**→ Report:** the relevant lines from `--help`/`config list` so we confirm the exact knob names before writing config. If a name below differs on this version, use the real one and note the substitution.

---

## 3. Create the brain (PGLite, isolated location)

Keep the brain out of `~/.hermes` so it is clearly a separate layer:

```bash
mkdir -p ~/gbrain-data && cd ~/gbrain-data
gbrain init                 # creates the PGLite DB here (no server)
gbrain doctor --json        # expect DB checks to pass; embeddings/keys will FLAG — that is expected for now
```

**→ Report:** full `gbrain doctor --json`. We expect green on DB/runtime and red/warn on embeddings + missing keys — that is the intended starting state, not a failure.

---

## 4. Configure DeepSeek as the chat provider (the tricky part)

GBrain reads chat creds from `OPENAI_API_KEY` and uses the OpenAI SDK, which honors a base-URL override. Point both at DeepSeek. **Do NOT put the DeepSeek key in the default Hermes env** — scope it to GBrain.

Create `~/gbrain-data/.env` (GBrain reads `.env` from its working dir):

```bash
# Reuse the DeepSeek key Hermes already has, without copying the secret by hand:
DS_KEY=$(grep -E '^DEEPSEEK_API_KEY=' ~/.hermes/.env | cut -d= -f2-)

cat > ~/gbrain-data/.env <<EOF
OPENAI_API_KEY=${DS_KEY}
OPENAI_BASE_URL=https://api.deepseek.com/v1
EOF
chmod 600 ~/gbrain-data/.env
```

Then set the chat model to a DeepSeek model via the **real** config key you found in §2 (likely one of these — verify):

```bash
gbrain config set search.mode conservative
# set the chat model to DeepSeek's — use the actual key name from `gbrain config --help`:
gbrain config set search.model deepseek-chat        # or deepseek-v4-pro (mirror Hermes' default)
# if a separate provider/base_url config key exists, set it too:
gbrain config set search.base_url https://api.deepseek.com/v1
```

**IMPORTANT — embeddings caveat:** Because `OPENAI_BASE_URL` now points at DeepSeek, any OpenAI *embedding* call will fail (DeepSeek serves no embeddings). That is fine **only because embeddings are disabled** (next step). Do not run `gbrain embed` until §7.

**→ Report:** the final `gbrain config list` (redact the key) so the operator can confirm chat=DeepSeek, mode=conservative, embeddings not active.

---

## 5. Import seed data WITHOUT embeddings, verify graph + lexical search

Start with a tiny, safe corpus to prove the pipeline. **Do not import secrets or the whole home dir.** Good first corpus: a copy of the existing memory notes (this is also the test data for the future consolidation loop).

```bash
mkdir -p ~/gbrain-data/seed
cp ~/.hermes/memories/MEMORY.md ~/gbrain-data/seed/   # known-messy on purpose
cp ~/.hermes/memories/USER.md   ~/gbrain-data/seed/

gbrain import ~/gbrain-data/seed/ --no-embed          # NO embeddings
gbrain extract links --source db --dry-run | head -20 # preview graph edges
gbrain extract links --source db                      # build the knowledge graph
gbrain extract timeline --source db
gbrain stats
gbrain query "what does the operator prefer before files are modified?"
```

A useful signal: GBrain's contradiction/dedup machinery should eventually notice the **two near-duplicate "Kimi context-window bug" entries** in `MEMORY.md`. Note whether `gbrain query` / `gbrain stats` surfaces them.

**→ Report:** `gbrain stats` and the `gbrain query` answer. If lexical+graph search returns sensible results, the core layer works without paying for embeddings.

---

## 6. Wire GBrain into an ISOLATED Hermes profile over MCP

Do **not** add the GBrain MCP server to the default `hermes` profile. Clone a dedicated one:

```bash
hermes profile create gbrain --clone     # isolated personality/session state
hermes profile alias gbrain               # per operator preference; avoid `profile use` (sticky default)
```

Register GBrain as an MCP server **for the `gbrain` profile only**. Use the absolute `gbrain` path from §1 and its working dir:

```bash
# stdio MCP — discover the exact registration command for this Hermes version first:
hermes mcp --help        # or: hermes profile gbrain mcp add ...
# Expected shape (adapt to the real CLI):
#   command: /abs/path/to/gbrain
#   args:    ["serve"]
#   cwd:     /home/agent/gbrain-data
```

If Hermes registers MCP servers via `config.yaml`, edit **only the `gbrain` profile's** config (under `~/.hermes/profiles/gbrain/`), never the root `~/.hermes/config.yaml`. Show the diff before saving.

**→ Report:** the exact MCP registration you made and the profile path it landed in. **Pause here for operator approval before first launch.**

---

## 7. (LATER — needs operator approval) Enable semantic embeddings + nightly dream cycle

Only after §1–6 are confirmed working and the operator approves spend:

- **Embeddings:** obtain a `ZEROENTROPY_API_KEY` (`ze-...`, GBrain's default embedder/reranker) OR a real `OPENAI_API_KEY`. Put it in `~/gbrain-data/.env` **without** the DeepSeek base-URL override clobbering it (embeddings and chat may need separate provider config — verify in §2's config surface). Then:
  ```bash
  gbrain embed --stale
  gbrain doctor --json     # embeddings should now pass
  ```
- **Recurring jobs (the loop / "dream cycle"):** register cron using **absolute paths**, scoped so it runs against `~/gbrain-data`:
  ```bash
  # every 15 min:  gbrain sync --repo ~/gbrain-data && gbrain embed --stale
  # nightly:       gbrain dream        # 8-phase: entity sweep, citation fixes, memory consolidation, contradictions
  # weekly:        gbrain doctor --json
  ```
  Prefer explicit cron over `gbrain autopilot --install` so the schedule is auditable. **Do not enable nightly `gbrain dream` until the operator confirms** — it is the part that spends DeepSeek tokens unattended (this is exactly the "loop running unattended is also a loop making mistakes unattended" risk).

---

## 8. Guardrails (standing rules for this install)

- Touch the **default `hermes` profile / root config: never.** All GBrain wiring lives in the `gbrain` profile + `~/gbrain-data`.
- **No unattended spend:** no `--auto`, no `--max-usd`, no `autopilot`, no nightly `dream`, no paid embeddings — until the operator explicitly approves each.
- **Secrets:** reuse `DEEPSEEK_API_KEY` by reference (§4); never echo key *values* into reports or logs.
- **Report-and-pause** at the end of §1, §3, §4, §5, and §6. Show command output / diffs; wait for approval before steps that affect the default profile or money.
- If any flag/key name in this guide doesn't exist on the installed version, use the real one from §2 and **note the substitution** in your report.

---

## 9. Definition of done (Phase 1)

- `gbrain --version` works; brain initialized in `~/gbrain-data` (PGLite).
- Chat configured to DeepSeek; `conservative` mode; embeddings off.
- Seed corpus imported `--no-embed`; graph + lexical `gbrain query` returns sensible answers.
- A `gbrain` Hermes profile exists with the GBrain MCP server registered (default profile untouched).
- Operator has all reports and has approved before any nightly/paid step.

> This GBrain instance is the **"relationships" layer** of the HiveMind stack (see `hivemind-loop-plan.md` §3). The next milestone is the **memory-consolidation loop**: a scheduled maker/checker loop that cleans `MEMORY.md` and projects curated facts into this graph — at which point the dirty seed data from §5 becomes the first real job.
