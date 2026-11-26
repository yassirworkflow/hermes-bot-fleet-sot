# 01 — CONTEXT

> Motivation, user base, boundaries. **Critical element: Non-Goals — what the agent must ignore.**

---

## 1. Document control

| Field | Value |
|---|---|
| Version | 1.0 |
| Status | Baseline for authorization |
| Date | 2026-11-17 |
| Source investigation | `../hermes-bot-fleet-investigation.md` |
| Authority | Owner (Yassir); agentic-engineer prepares, Owner authorizes per phase |
| Change rule | Any change to scope, model policy, roster, or credentials requires a new version + re-authorization |

---

## 2. Motivation (problem statement)

The local Hermes Desktop install hosts a fleet of agent profiles ("Bots") that has grown organically. The supplied inventory was materially wrong (34 real profiles vs 32 claimed; a missing estate bot; a ghost `researcher` profile; skill/plugin counts off by up to 90×). Beyond the inventory defect, five structural problems were confirmed:

1. **Governance blindness (R1).** No authoritative registry. Decision-making relies on an inaccurate roster; one profile (`researcher`) has neither `config.yaml` nor `profile.yaml`.
2. **Policy drift (R2).** `LOCAL_MODEL_POLICY.md` mandates a local-first architecture (9B/7B/2.6B on oMLX), but 24 profiles inherit a global OpenRouter `nemotron-3-ultra-550b:free` pin and 5 more are pinned to other models. The fleet is cloud-first in practice: cost, latency and data-egress exposure.
3. **Selector blocked (R3).** The stated goal — every bot governed only by the native model-selector button — is impossible while `model.default`/`model.provider` pins exist in 10 profile configs plus the global config. Docs confirm the profile pin "takes over again" over a chat-level selector choice.
4. **Role/persona mismatch (R4).** 22 role descriptors exist in `~/.hermes/souls/` but only `asset-manager` has its role intent, KPIs and approval gates actually applied to its SOUL.md. 30+ bots run the generic Hermes persona; no operational guardrails.
5. **Credential & capability sprawl (R5, R6, R7).** Eight profile `.env` files duplicate plaintext API keys; the guardian backup profile loads 2,251 skills; toolsets are broad by default (browser, computer_use, image_gen, delegation, cronjob on almost every bot).

**Consequence of inaction:** unreliable bot behaviour, unpredictable spend, no auditable guardrails for finance/operations roles, and group/bot-to-bot features layered on a faulty foundation.

---

## 3. Vision & objectives

**Vision:** a self-contained, auditable team of specialist Bots on one Hermes instance, each with a correct role identity, minimal safe capabilities, selector-driven model control, and reliable department-level collaboration.

| ID | Objective | Measure |
|---|---|---|
| O1 | One authoritative bot registry matching disk | 34/34 profiles enumerated; zero ghosts/unknowns |
| O2 | Every bot's SOUL matches its title/role | 100% of active bots have applied role SOUL with guardrails |
| O3 | Native selector is sole main-model control | 0 profile-level main-model pins; global default set via native tooling |
| O4 | Predictable cost/quality | All auxiliary slots on validated 2.6B local model; documented switch hygiene |
| O5 | Least-privilege capabilities | Every bot has an explicit tool allowlist + role-relevant skill subset |
| O6 | Credential hygiene | Shared secrets only in root `.env`; profile `.env` only for profile-specific secrets |
| O7 | Department collaboration works | 4 group rooms (2–6 bots each) with 3-round/10-message caps verified |
| O8 | Bot-to-bot delivery is reliable & diagnosable | Fire-and-forget ack, retry-once, typed reason codes observed in tests |

---

## 4. User base & stakeholders

| Actor | Role | Needs |
|---|---|---|
| Owner (Yassir) | Operator / approver | Cost control, guardrails on capital/external comms, auditability, phase authorization |
| Assistant agent (this program) | Engineer / reviewer | Discover, design, verify, roll back — never mutate without authorization |
| Core bots | orchestrator, dev-coder, researcher-bot | Dispatch, coding, evidence-led research |
| HCREALTY bots (19) | Hospitality estate team | Department coordination (F&B, Sales & Marketing, Strategic Board), role-specific reasoning |
| AI-infra bots (3) | local-ai-doctor, guardians | Diagnostics, observability, safe operation |
| Specialist bots (6) | debugger, reviewer, planner, researcher, deputy-agent, ape-bot | Narrow delegated tasks |
| Other bots (3) | splash-7777, competition-survey-manager, culinary-frontline-agent | Retire-or-keep decision in P0 |

---

## 5. Current state (as-is) — evidence summary

| Dimension | As-is | Evidence |
|---|---|---|
| Profile count | 34 directories | `ls ~/.hermes/profiles/` |
| Config shape | 10 `config.yaml`, 23 `profile.yaml`-only, 1 neither | direct inspection |
| Global main model | `nvidia/nemotron-3-ultra-550b-a55b:free` @ openrouter | `config.yaml` line ~483 |
| Policy | local-first 9B/7B/2.6B, no cloud fallback | `LOCAL_MODEL_POLICY.md` |
| SOULs | 22 role descriptors in `~/.hermes/souls/`; 1 applied | `souls/` vs profile SOUL.md diff |
| Skills | global 2,273; profile 12–45; backup 2,251 | directory counts |
| Credentials | 8 profile `.env` with duplicated keys | key-name scan |
| Bot-Mode marker | only `orchestrator/profile.yaml` has `ui_meta: {hermes-bots: {}}` | profile.yaml read |
| Canonical Bot Chats | unverified per bot | needs authorized `hermes -p <bot> chat -c "Bot Chat" --create-if-missing` |
| Cron/routines | runtime artifacts present, job definitions unverified | `cron/` dirs; needs `hermes cron list` |

**Archived external reference:** `NousResearch/Hermes-Bot-Mode` is archived (2026-08-17); Bot Mode is built-in to Desktop. Design against official docs at `hermes-agent.nousresearch.com/docs`, not the archived plugin repo.

---

## 6. Scope

### 6.1 In scope (authorized program)

- Authoritative registry and profile normalization (create/fix `profile.yaml`, resolve ghost/backup/retire decisions).
- SOUL/AGENTS/memory minimum configuration per bot, derived from `souls/` descriptors.
- Capability curation: per-profile tool allowlists, skill subset, plugin decisions.
- Model governance: unset all main-model pins; native-selector-only; auxiliary slots to 2.6B local; policy amendment.
- Credential consolidation to root `.env` with `${VAR}` / `key_env` references.
- Four group rooms (HCREALTY F&B, HCREALTY Sales & Marketing, HCREALTY Strategic Board, CODE/DEV) with documented caps.
- Bot-to-bot messaging verification (message_agent, @mentions, retry-once, typed reason codes).
- Verification evidence, snapshots and rollback per phase.

### 6.2 Out of scope (hard boundary)

- No implementation of new application code, plugins, skills, or schema.
- No edits to `~/.hermes/` until the specific phase is authorized.
- No dependency changes, no `hermes update`, no gateway version changes.
- No commits, PRs, merges, deployments, or production write-back.
- No changes to `hcrealty-kb` content, investment models, PIPELINE.md/ASSETS.md/COMP-SET.md data.
- No capital commitment, LOI signing, financial transaction, or external communication by any bot.

---

## 7. Non-Goals — what the agent MUST ignore

> These are explicit "do not do" instructions. If a future step appears to require one of them, stop and escalate rather than proceed.

1. **Do not build a new agent framework.** Bot Mode is built into Hermes Desktop; there are no core patches and no new primitives. A Bot *is* a profile.
2. **Do not force a single model onto every bot via config pins.** Main-model pins are being *removed*, not standardized. Selector-driven per-chart choice is the target. Per-role model assignment via config is deliberately abandoned (see Architecture invariant A-INV-4).
3. **Do not delete or archive any profile without explicit Owner approval.** Including `researcher`, `splash-7777`, and `local-ai-guardian.backup.20261103-142924`. Retirement is a decision, not a default.
4. **Do not rewrite the global `~/.hermes/SOUL.md`** (custom "Master-Chief-Orchestrator") unless Phase 0 explicitly rules on identity-layer precedence. Profile SOULs are the target of this program.
5. **Do not enumerate, install, or "fix" the 2,273-skill global catalog.** Only role-relevant profile subsets are in scope. Do not create new skills.
6. **Do not migrate `hcrealty-kb`, pgvector, SearXNG, oMLX, llama-swap, or Splash infrastructure.** They are treated as available services, not work items. Splash (:7777) is known dead and is not to be revived here.
7. **Do not enable unattended data-training tiers or remove approval gates.** `-contributor` models and `security.allow_data_training_tiers_noninteractive` stay off.
8. **Do not read, copy, print, or relocate secret values.** Work with variable *names* and references only; secret values never enter reports, prompts, or logs.
9. **Do not change hardware/model topology** (no 27B VLM/MTP; the 48 GB memory ceiling and validated 9B/7B/2.6B set stand).
10. **Do not treat the archived Bot Mode plugin repo as authoritative.** Ignore its install instructions; use built-in Bot Mode docs.
11. **Do not perform mid-session model switches in verification tests unless testing the warning path** — they break prompt cache and distort cost measurements.
12. **Do not auto-approve destructive actions on behalf of bots.** `asset-manager` and any finance/ops role keep `confirm_destructive: true` and Owner-approval rules.

---

## 8. Authorization register (to be completed by Owner)

| Phase | Scope authorized | Date | Decision |
|---|---|---|---|
| P0 | Adopt 34-bot registry; keep/retire decisions; ratify or replace `LOCAL_MODEL_POLICY.md` | pending | — |
| P1 | Read-only validation commands | pending | — |
| P2 | SOUL/AGENTS/memory role alignment + capability curation | pending | — |
| P3 | Unset model pins + native selector + aux models | pending | — |
| P4 | Group rooms + bot-to-bot verification + credential hygiene | pending | — |
| P5 | Final verification, evidence, rollback authority | pending | — |

No phase is implied by another. Each requires a separate "authorized" entry.

---

## 9. Constraints

| Constraint | Detail |
|---|---|
| Hardware | 48 GB Apple M4 Pro; OS + oMLX ~8 GB; concurrent 9B+7B+2.6B ~10 GB; 27B VLM excluded |
| Local endpoints | oMLX `:8888` (shared), llama-swap `:8085` (researcher-bot exception), SearXNG `:8282`, Splash `:7777` (dead) |
| Docs authority | `hermes-agent.nousresearch.com/docs` (Bot Mode, Configuration, Configuring Models, Which File Does What) |
| Config precedence | CLI args > `config.yaml` > `.env` > built-in defaults |
| Prompt cache | SOUL = slot #1; MEMORY/USER = frozen snapshot at session start; mid-session model switch forces full re-read |
| Group rooms | 2–6 bots; 3 serial rounds; 10 messages/send; ~200 messages / 32,000 chars transcript; 8,000-char oversized excerpt |
| Loop guard | `gateway.bot_loop_guard.max_hops: 4`; `bot_replies_expect_reply: false` |

---

## 10. Glossary

| Term | Meaning |
|---|---|
| Bot | A Hermes profile exposed in the Bots pane; isolated config, memory, skills, credentials, sessions |
| Canonical Bot Chat | Session titled exactly `Bot Chat`; required for `message_agent` injection |
| Bot-Mode marker | `ui_meta: { hermes-bots: {} }` in any `profile.yaml`; marks the install |
| Main model | Model used for thinking/tool loops; selector-controlled after P3 |
| Auxiliary model | Side-job model (compression, titles, approval, vision, group router, …) |
| Pin | `model.default` + `model.provider` in a profile config; overrides selector |
| Typed reason code | Machine-readable failure code on delivery (e.g. `provider_rate_limit`) |
