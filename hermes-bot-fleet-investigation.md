# Hermes Desktop Bot Fleet — Discovery, Risk & Proposed Scope

**Mode:** Investigation only. No files under `~/.hermes/` were modified. No commits, PRs, deployments, dependency/schema changes, or production write-back.
**Date:** 2026-11-17
**Evidence base:** direct inspection of `/Users/user/.hermes/profiles/` (34 dirs), global `~/.hermes/config.yaml`, `LOCAL_MODEL_POLICY.md`, `~/.hermes/souls/`, plugin/skill catalogs, and official docs (Bot Mode, Configuration, Configuring Models, Which File Does What). Tech doc repo `NousResearch/Hermes-Bot-Mode` is **archived (2026-08-17)**; Bot Mode now ships built-in — the supplied inventory is partly based on the obsolete plugin model.

---

## 1. Discovery findings

### 1.1 Inventory challenge — the supplied inventory is materially wrong

| Claim in supplied inventory | Actual on disk | Verdict |
|---|---|---|
| "32 profiles (including backup)" / "30 profiles/bots" | **34 profile directories** | Undercount |
| HCREALTY = 18 bots | **19 bot dirs** | Missing `estate-housekeeping-agent` |
| Summary table: "Other = 2" but lists 3; totals 32 vs 33/34 | Internal arithmetic errors | Unreliable |
| All profiles have `config.yaml` | **Only 10** have `config.yaml`; 23 have `profile.yaml` only; `researcher` has **neither** | Wrong |
| `researcher-bot`: 7 skills, no plugins | **21 skills, 1 plugin** (`orca-status`) | Wrong |
| `dev-coder`: 25 skills, 3 plugins | **20 skills**, config enables **5** plugins (3 dirs) | Wrong |
| `local-ai-doctor`: 35 skills, 2 plugins | **22 skills**, config enables 2 plugins (0 dirs) | Wrong |
| `local-ai-guardian`: 25 skills; backup: 40 skills | **45** and **2,251** skills respectively | Wrong by ~90x for backup |
| `orchestrator` memory: MEMORY.md + USER.md | Only **MEMORY.md, 38 bytes** ("Active Hermes profile: researcher-bot"); **no USER.md anywhere** | Wrong |
| All HCREALTY bots have "custom SOUL.md for each role" | All 19 use the **generic Hermes default SOUL** except `asset-manager` | Wrong |
| "190+ skills loaded" | Global catalog has **2,273** skill dirs; a given profile loads **12–45** (backup: 2,251) | Conflated global catalog with profile load |

**Bot missing from inventory:** `estate-housekeeping-agent` — real Bot Mode profile (`profile.yaml`: `role: general`, version 1), with a role descriptor in `~/.hermes/souls/`, 12 base skills, and its own cron/state DB.

**Anomalies:** `researcher` has no `config.yaml` and no `profile.yaml` (ghost/empty profile). `local-ai-doctor` and `splash-7777` have `config.yaml` but no `profile.yaml`. `profiles/.deleted/` is empty.

**Corrected count:** Core 3 + HCREALTY 19 + AI-infra 3 + Specialists 6 + Other 3 = **34**.

### 1.2 Actual per-profile configuration

Model pins (profiles with `config.yaml`):

| Profile | Pinned main model | Provider |
|---|---|---|
| orchestrator | `nvidia/nemotron-3-ultra-550b-a55b:free` | openrouter |
| researcher-bot | `nvidia/nemotron-3-ultra-550b-a55b:free` | openrouter |
| asset-manager | `nvidia/nemotron-3-ultra-550b-a55b:free` | openrouter |
| culinary-director | `qwen3.6-plus-free` | opencode |
| dev-coder | `mlx-community--Qwen3.6-35B-A3B-4bit` | omlx |
| splash-7777 | `mlx-community--Qwen3.6-35B-A3B-4bit` | omlx |
| ape-bot | `MLX-Qwen3.5-9B-DeepSeek-V4-Flash-4bit` | omlx |
| local-ai-doctor | `MLX-Qwen3.5-9B-DeepSeek-V4-Flash-4bit` | omlx |
| local-ai-guardian (+ backup) | `MLX-Qwen3.5-9B-DeepSeek-V4-Flash-4bit` | omlx |

The other **24 profiles have no model pin** and therefore inherit the **global** `config.yaml`: `nvidia/nemotron-3-ultra-550b-a55b:free` on OpenRouter. That includes all 18 generic HCREALTY bots and all specialists (`debugger`, `reviewer`, `planner`, `researcher`, `deputy-agent`, …). So the fleet is **cloud-first in practice**, despite the local-first policy document.

Skill / plugin / memory reality (selected):

| Profile | Skills | Plugins (dirs) | Memory | SOUL |
|---|---|---|---|---|
| orchestrator | 17 | 0 | MEMORY.md 38 B | Custom dispatcher |
| dev-coder | 20 | 3 (config: 5) | none | Custom coding |
| researcher-bot | 21 | 1 (`orca-status`) | 2 files | Custom 8.3 KB research persona |
| asset-manager | 20 (12 base + 8 `.md`) | 0 | KB files via config | Custom full role SOUL |
| local-ai-doctor | 22 | 0 (config: 2) | none | Custom diagnostician |
| local-ai-guardian | 45 | 4 | 2 files | Custom |
| guardian backup (2026-11-03) | **2,251** | 8 | 1 file | Custom |
| all other 24 profiles | **12 identical base skills** | 0 | 0 | Generic default Hermes SOUL |

### 1.3 SOUL / role alignment gap

- `~/.hermes/souls/` holds **22 role descriptors** (asset-manager, gm-agent, estate-accountant, hammam-master, etc.). Their frontmatter is `name / description / category` (skill-style metadata), **not** SOUL prompt bodies. They encode real role intent, KPIs and guardrails (e.g. asset-manager: "Never commit capital/sign LOIs without Owner approval").
- Only **`asset-manager`** has those rules actually written into its profile `SOUL.md`. The other 18 HCREALTY bots run the generic "You are Hermes Agent, built by Nous Research…" persona with **no role instructions, no KPIs, no approval gates**.
- `culinary-director` uses the generic SOUL despite having `use_real_profile: true` browser config and two API keys.
- The global `~/.hermes/SOUL.md` is itself a custom "Master-Chief-Orchestrator" persona — so there are three competing identity layers (global SOUL, profile SOUL, souls library descriptors).

### 1.4 Model-policy drift

`LOCAL_MODEL_POLICY.md` declares a canonical local-first assignment. Actual state contradicts every row:

| Profile | Policy says | Actual | Status |
|---|---|---|---|
| orchestrator | 9B local oMLX | OpenRouter Nemotron 550B free | DRIFT |
| default (global) | 9B local oMLX | OpenRouter Nemotron 550B free | DRIFT |
| dev-coder | 7B coder local | 35B local oMLX | DRIFT |
| researcher-bot | `qwen3.5-4b` llama-swap | OpenRouter Nemotron 550B free | DRIFT |
| local-ai-doctor | Nous `solar-pro4:free` (exception) | 9B local oMLX | DRIFT |

Global config also contradicts policy: `model.provider: openrouter` while `openrouter.default_provider: omlx` and `fallback_providers: omlx`; policy §5 mandates `explicit_only` fallback with no cloud fallback.

### 1.5 Native model-selector mechanic (documented, authoritative)

Official Bot Mode docs: *"Model & provider pin — give the Bot its own model… **Leave it unset to inherit from the launch profile.** Picking a model from the Bot Chat's composer sticks to that chat (it survives reopening the app) **until you change the Bot's profile model, which takes over again.**"*

Therefore: **every profile-level `model:` pin must be unset (or removed) before the native selector can be the sole control.** The global `model:` pin must also be cleared or deliberately set via `hermes model`/dashboard. Mid-session switches break the prompt-cache prefix (full re-read of context at full input price); docs advise switching early or using `--once` only for short sessions. Config changes apply to **new sessions only**; `/model` hot-swaps the current chat.

### 1.6 Tools / skills / credentials

- Only 10 profiles define a tool allowlist style config; `asset-manager` alone has explicit `tools:` + `permissions:` (read/write/execute/network true, delete ask). Everything else uses defaults or broad `platform_toolsets` lists (browser, computer_use, image_gen, delegation, cronjob, discord, homeassistant, etc.) — far beyond most roles.
- `local-ai-guardian.backup` loading **2,251 skills** is an outlier; the global catalog is 2,273 skills. Duplicate skills exist (`systematic-debugging` ×3, `verification-before-completion` ×2).
- 8 profile `.env` files carry plaintext API keys. `orchestrator`, `local-ai-doctor`, both guardians duplicate OPENROUTER/OPENAI/FREELLMAPI/OPENCODE_ZEN keys; orchestrator and doctor also carry Telegram/Buzz relay credentials. This violates policy §6 (shared keys belong in root `.env`; profile `.env` only for profile-specific secrets).

---

## 2. Risk assessment

| ID | Risk | Evidence | Severity |
|---|---|---|---|
| R1 | Inventory/governance integrity — decisions based on wrong roster; ghost `researcher`; missing estate bot | 34 vs 32; no config/profile for `researcher` | High |
| R2 | Policy drift → cloud cost, latency, data egress | 24 unpinned bots + 5 pinned bots contradict `LOCAL_MODEL_POLICY.md` | High |
| R3 | Native selector goal blocked by pins | `model.default`/`provider` set in 10 profile configs + global | High (blocks stated objective) |
| R4 | Persona/role mismatch & missing guardrails | 30+ bots on generic SOUL; 22 role descriptors unused; financial/operational approval gates absent outside asset-manager | High |
| R5 | Credential sprawl | 8 profile `.env` files with duplicated plaintext keys | High |
| R6 | Context/token bloat from skill loadout | backup 2,251 skills; global 2,273; duplicate skills | Medium–High |
| R7 | Over-broad toolsets / permissions | default `platform_toolsets` include browser/computer_use/image_gen/delegation for all; asset-manager write+execute+network | Medium |
| R8 | No USER.md; memory stub only | orchestrator MEMORY.md = 38 B | Low–Medium |
| R9 | Cron/routines unverified | every profile has cron runtime artifacts, no job definitions confirmed; needs authorized `hermes cron list` | Medium |
| R10 | Stale backup profile & archived-plugin assumptions | backup dated 2026-11-03; Hermes-Bot-Mode repo archived | Low–Medium |

**No risk was remediated** — all are findings for the authorization decision.

---

## 3. Recommendations (for the authorization decision, not for execution now)

### 3.1 Bots configured to role/title
- Establish the **discovered 34-profile registry** as the single source of truth; resolve `researcher` (retire or configure), decide fate of `local-ai-guardian.backup…` and `splash-7777`.
- Give every bot a `profile.yaml` with accurate `name`, `description`, `role` (Bot Mode uses description/role in teammate rosters and group routing).
- Ratify `LOCAL_MODEL_POLICY.md` as canonical **or** formally replace it with a cloud-first policy — the current hybrid is the root of R2.

### 3.2 SOUL redaction to intent
- Source each role SOUL from `~/.hermes/souls/<bot>-soul.md` descriptors; write concise role identity + explicit guardrails (Owner approval for capital/LOI/external comms; evidence/citation requirements; escalation to GM).
- Keep SOUL.md lean — it is slot #1 of the system prompt on every session. Remove generic boilerplate where a role persona is intended; keep the default persona only for genuinely generic helper bots.
- Resolve the three identity layers: decide precedence of global SOUL vs profile SOUL vs souls library.

### 3.3 Tools — keep / delete / missing
- **Keep:** `file`/`terminal` for dev-coder & local-ai-doctor; `web`/`browser` for researcher-bot & culinary-director; `delegation`/`message_agent` for orchestrator & gm-agent; `memory`/`session_search`; deferred `tool_search` (already on).
- **Restrict/delete:** browser, computer_use, image_gen, cronjob, discord_admin, homeassistant, kanban, x_search on guest-facing/finance-adjacent and leaf specialist bots; `delegation` on leaf bots; write/execute/network on read-only roles.
- **Missing:** per-profile tool allowlists (only asset-manager has one); role-specific skills for the 18 estate bots (hospitality KPIs, RevPAR/ADR/GOPPAR, food cost, beverage cost, housekeeping/laundry QA, spa/hammam SOPs); USER.md; group-router auxiliary model.
- Archive/remove the 2,251-skill backup profile and dedupe skills after approval.

### 3.4 Quality/output optimization
- Right-size models by task: local 9B/7B for engineering, 2.6B for auxiliary, cloud only for heavy synthesis with explicit approval.
- Enforce evidence-ledger/citation output contracts for research and asset-manager roles; verification-before-completion for engineering.
- Dedupe skills; cap enabled skills per profile to role-relevant sets; keep memory (frozen snapshot) small and current.
- Use `auxiliary.group_router` (cheap model) for group-chat auto-assignment.

### 3.5 Token/cost control without quality loss
- Pin all auxiliary slots (compression, title_generation, approval, vision, web_extract, mcp, skills_hub) to the validated 2.6B local model per policy §4 — dev-coder currently pins them to 35B; guardian/global partly unset.
- Switch models **early** in a conversation; avoid frequent mid-session switches and `--once` on long sessions (each switch re-reads the full uncached context).
- Use `/compact` for long Bot Chats rather than forking; keep SOUL and MEMORY.md terse.
- Let `tool_search` defer the 67+ tools instead of loading all toolsets.

### 3.6 Native selector-only setup (target end-state)
1. Remove/unset `model.default` and `model.provider` in all 10 profile `config.yaml` files (and clear the global `model` pin via `hermes model` / dashboard to the intended default).
2. Leave profile model pins unset so bots **inherit from the launch profile**; then per-chat selector choices persist across app restarts.
3. Configure auxiliary slots explicitly to small local models so selector switches don't drag side-jobs onto an expensive model.
4. Verify with `hermes config get model --json`, `hermes config check`, and a per-bot smoke test that the selector sticks after reopening the app.

---

## 4. Proposed scope for a separate authorization decision

**Out of scope now (unchanged):** all file edits under `~/.hermes`, deletions, config writes, dependency/schema changes, production write-back, deployment, commits/PRs.

**Request authorization for:**

| Phase | Decision requested | Artifacts / gates |
|---|---|---|
| 0. Governance | Adopt discovered 34-bot registry as source of truth; decide keep/retire `researcher`, `splash-7777`, guardian backup; ratify or replace local-model policy | Written decision record |
| 1. Read-only validation | Permit `hermes cron list`, `hermes profile list`, `hermes config get model --json`, per-profile model/skill audit | No writes; outputs to report only |
| 2. Persona alignment | Approve role-SOUL templates for 19 estate bots + 6 specialists + role-specific skills; define approval gates | Backup SOULs before change; diff review |
| 3. Selector migration | Approve unsetting all model pins + global model set via native tool; aux models to 2.6B | Snapshot configs; staged canary (one bot) then fleet |
| 4. Tool/credential hygiene | Approve per-profile tool allowlists, skill dedupe/removal, consolidate duplicated keys to root `.env` | Secret handling plan; verify no key leakage |
| 5. Verification | Approve smoke tests, `hermes doctor`/`config check`, rollback criteria | Evidence report + rollback snapshots |

Each phase should be authorized separately; no phase is implied by this investigation.
