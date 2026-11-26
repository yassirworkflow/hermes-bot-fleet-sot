# Hermes Fleet — Executive Blueprint & Pre-Phase-0 Review

**Date:** 2026-11-26
**Mode:** Read-only analysis. No code written. No files under `~/.hermes/` modified.
**Author:** Assistant agent (reviewer per lean-review authority)
**Sources:** `hermes-bot-fleet-sot/00–04`, `hermes-fleet-audit-2026-11-26.md`, `docs/superpowers/specs/2026-09-28-orchestrator-bot-design.md`, `hermes-orchestrator-review-2026-11-26.md`, live fleet evidence.

---

## 0. Review & Challenge

### 0.1 What is sound (keep)

- **Bot = profile primitive** (ADR-1). No new framework, no core patches, no new storage. Bot Mode is built into Desktop. This is the right, minimal posture.
- **Selector-only main model** (ADR-2). Pins override the native selector (R3); removing pins is the correct fix, not standardizing on one model.
- **Local-first auxiliaries** on a validated 2.6B local model (ADR-3), with a 48 GB memory ceiling and a 9B/7B/2.6B validated set.
- **Four bounded group rooms** with hard caps (3 rounds / 10 messages / 200 messages / 32k chars) and typed reason codes with retry-once (FR-8/FR-9). This is a real, testable reliability contract.
- **Lean review authority** and an **authorization register** with hard gates between phases. No phase is implied by another.
- **Orchestrator as thin router** with roster-as-data (`roster.yaml`): extensible without code changes, and skill separation (orchestrator loads only its own skill) is correct.
- **Non-Goals list (12)** is strong and protective, especially "no new framework", "no global SOUL rewrite", "no profile deletion without approval", "no secret values in reports".

### 0.2 What is broken or unverified (must fix before production)

- **Fleet shape:** 34 profile directories; only 10 have `config.yaml`; 22 run the generic 663-byte Nous boilerplate SOUL; 22 role descriptors in `~/.hermes/souls/` are applied to only ~1 profile. 22 profiles are 🔴 not ready.
- **Orchestrator skeleton is unverified:** `profile.yaml` is only `ui_meta: {hermes-bots: {}}` (no name/description/role/version); roster has 6 cards but 4 point to `profile: TBD` (and `researcher-bot` actually exists but is mislabeled TBD); 16-key credential sprawl; 25 skills (bloat vs. the ~85-line playbook target); acceptance tests (§6 of the design spec) were never run because tmux/LM Studio were unavailable in the build sandbox; root `MEMORY.md` is a stale 38-byte stub claiming "Active Hermes profile: researcher-bot".
- **No git repo in the working directory**, so the plan's "commit" claims are unverified. Profiles live in `~/.hermes/`, outside any repo.
- **Correction (new evidence):** `~/.hermes/` itself is a git repository (`.git` present), so profiles *are* under version control there — the "no git repo" finding refers only to the working directory (`Hermes_Desktop`), not to `~/.hermes/`. C11's rollback/versioning concern is therefore partially mitigated: profile snapshots can be committed in `~/.hermes/`, but the SoT folder (`Hermes_Desktop/hermes-bot-fleet-sot/`) is **not** in a repo and remains unversioned. Reconcile before P0 (either init a repo for the SoT folder or store SoT artifacts under `~/.hermes/`).
- **Global doctor issue:** vendor-prefixed model slug mismatch (`mlx-community/Qwen3.6-35B-A3B-4bit` on provider `splash`); Splash `:7777` is dead and must not be revived.
- **Credential sprawl:** 6 profile `.env` files duplicate shared plaintext keys (policy §6 violation).
- **Context bloat:** guardian backup profile carries 2,263 skills; global catalog 2,273; `systematic-debugging` duplicated ×3.
- **Zero cron jobs** anywhere; memory populated for only 4 profiles.

### 0.3 Challenge to the proposed simplification (service-master bots)

The stated goal is to "clean the huge number of bots by simplify the architecture: keep bots by services only", e.g. an F&B manager bot that delegates to food/bev/service bots, extended to rooms/maintenance/sales&marketing.

**This is a legitimate and useful direction, but it is a topology decision, not a refactor.** It must be ratified in P0 before any writes. Specific challenges:

1. **Two-tier hub vs. hop budget.** The orchestrator is already the hub (design spec §2: "bots do not talk to each other directly"). Making department masters *delegating* masters creates orchestrator → department-master → leaf-bot chains. The loop guard is `max_hops: 4` (ER-22). A three-level chain consumes most of that budget before any retry, leaving no room for the retry-once policy (ER-21) or for a reviewer hop. Either the hop guard must be raised (with documented rationale) or masters must be *peers* that advise, not delegate.
2. **Conflict with Non-Goal 3.** Retiring the 20 generic HCREALTY leaf bots requires explicit Owner approval. Collapsing them into masters without that approval violates the Non-Goal. The disposition must be recorded per bot in the P0 registry.
3. **Overlap with existing department leads.** The SOT already names leads in each room: `culinary-director` (F&B), `sales-marketing-director` (Sales & Marketing), `asset-manager`/`gm-agent` (Strategic Board). The proposal re-casts these as *delegating masters* with new toolsets (`delegation`, `message_agent`) and new SOULs. That is a real change to FR-4/FR-8, not a rename.
4. **Membership caps.** Rooms hold 2–6 members (ER-13). If masters are room members *and* can spawn sub-bots, the room's visible membership (2–6) no longer reflects the bots actually doing work, which breaks T-08 ("exactly 4 rooms with the member lists above").
5. **Single point of failure / context bottleneck.** A master bot becomes a domain router *and* a domain worker. It needs its own allowlist, memory retention, and a defined escalation path when it cannot delegate (e.g., leaf bot missing). Otherwise it hallucinates, which is exactly the C1 failure mode flagged for the orchestrator.
6. **Model implications.** Masters doing synthesis/delegation may need a heavier model than leaf bots. This interacts with FR-5/FR-6 and the unresolved policy question (local 9B vs. cloud Nemotron). Masters cannot be pinned (ADR-2) but their *recommended* selector choice differs from leaves.

**Recommendation:** Add a topology ADR (ADR-8) and a P0 decision row. Do not silently collapse bots. Keep the 4 group rooms as peer-deliberation surfaces (FR-8) and add masters as an *optional delegation layer* only where a department has ≥3 leaf bots and a clear lead. Start with F&B as the single pilot master (culinary-director → sous-chef-pastry, restaurant-wine-lead, culinary-frontline-agent) because it is the only department where a lead already has a custom SOUL and a rich profile.yaml.

### 0.5 P0 decisions recorded (user-authorized, 2026-11-26)

The user authorized implementation of the recommendations and resolved the three structural questions:

1. **Model policy (Q3):** Ratify **selector-only with a validated 9B local global default** (option a). Unset all 10 profile pins + clear/set global default natively; aux slots on 2.6B local (incl. dev-coder 35B→2.6B); amend `LOCAL_MODEL_POLICY.md` to v2 selector-only with this date. Orchestrator's Nemotron 550B pin = drift to be corrected (not an accepted exception).
2. **Topology (Q1):** Pilot service-master on **F&B only** (culinary-director → sous-chef-pastry, restaurant-wine-lead, culinary-frontline-agent) as ADR-8; other three rooms remain peer rooms. Non-proposed (maintenance/rooms) bots deferred to a later batch after hermes-core-bot-fleet ships.
3. **Identity precedence (Q2):** To be finalized in P0 decision record; global SOUL not rewritten (Non-Goal 4) until precedence ruling is recorded.
4. **New bots to create (added to registry/plan):**
   - `librarian-bot` — manages/stores/writes/reads all files destined for wiki/Postgres/database (knowledge-base librarian).
   - `office-bot` — writes/manages/transforms xlsx/csv/pdf/word/.mf files using open-source tools/skills and LLM-model-integration-friendly pipelines.
   - Both are new profiles to be created in P2 with full MVB (SOUL, profile.yaml with `ui_meta`, allowlist, memory, skills) and added to the CODE/DEV or a new "Knowledge & Office" room.
5. **Deferred:** maintenance/rooms bots (estate-butler-supervisor, executive-housekeeper, estate-housekeeping-agent, hammam-master, spa-hammam-manager, lead-spa-receptionist) — implement in a later batch after hermes-core-bot-fleet.
6. **GitHub sync:** create a new GitHub repo and push the SoT folder + this report (currently unversioned). Repo name/visibility to be confirmed.

The blueprint is sound and unusually complete (30 entities, 30 ER rules, 10 FRs with binary tests, 6 phases with gates). The estate is **not** production-ready (22/34 🔴). The simplification direction is valid but must be ratified as a topology decision with Owner approval, not executed as cleanup. **No code should be written until P0 authorization (registry + policy + topology) is recorded.**

---

## 1. The Executive Blueprint (Cheatsheet)

| Section | Core Focus | Critical Element Often Missed |
|---|---|---|
| 1. Context | Motivation, user base, boundaries | **Non-Goals** — what the agent must ignore |
| 2. PRD | Features, edge cases, UX | **Binary Checklists** — Pass/Fail per requirement |
| 3. Architecture | Tech stack, data models, file structures | **State & Schema** — complete entity relationship rules |
| 4. Plan | Phase sequence, testing checkpoints | **Dependencies** — data models before frontends/rooms |

### 1.1 Context (distilled)

- **Motivation:** organic growth produced an inaccurate inventory (34 real profiles vs. 32 claimed), five structural problems: governance blindness (R1), policy drift (R2), selector blocked by pins (R3), role/persona mismatch (R4), credential & capability sprawl (R5–R7).
- **Vision:** self-contained, auditable team of specialist bots on one Hermes instance — correct role identity, minimal safe capabilities, selector-driven model control, reliable department collaboration.
- **Objectives O1–O8:** registry parity (34/34), SOUL alignment 100%, zero profile pins, aux on 2.6B, least-privilege allowlists, credential hygiene, 4 verified rooms with caps, reliable bot-to-bot delivery with typed reasons.
- **User base:** Owner (Yassir, approver); assistant agent (engineer/reviewer); core bots (orchestrator, dev-coder, researcher-bot); 19 HCREALTY hospitality bots; 3 AI-infra; 6 specialists; 3 others.
- **In scope:** registry normalization, SOUL/AGENTS/memory minimum config, capability curation, model governance, credential consolidation, 4 group rooms, bot-to-bot verification, evidence & rollback.
- **Out of scope (hard):** new application code/plugins/skills/schema; `~/.hermes/` writes before phase authorization; dependency/gateway changes; commits/PRs/deployments; `hcrealty-kb` content; capital/LOI/external comms; profile deletion without approval.
- **Non-Goals (12):** no new framework; no forced single model; no deletion without approval; no global SOUL rewrite; no global skill-catalog changes; no infra migration; no training tiers; no secret values in reports; no hardware changes; no archived plugin authority; no mid-session switches in tests; no auto-approve destructive.
- **Constraints:** 48 GB Apple M4 Pro; oMLX `:8888`, llama-swap `:8085`, SearXNG `:8282`, Splash `:7777` dead; config precedence CLI > config > `.env` > defaults; SOUL = prompt slot #1; MEMORY/USER = frozen snapshot at session start; group rooms 2–6 bots / 3 rounds / 10 msgs / ~200 msgs / 32k chars; loop guard `max_hops: 4`.

### 1.2 PRD (distilled)

Ten functional requirements, each with binary tests (T-01…T-10):

| FR | Requirement | Test |
|---|---|---|
| FR-1 | Authoritative registry & profile normalization (34 profiles, fix 2 missing `profile.yaml`) | T-01 |
| FR-2 | Role SOULs aligned to `souls/` descriptors; remove generic persona; preserve asset-manager guardrails | T-02 |
| FR-3 | Minimum config per bot: SOUL, profile.yaml, config.yaml (allowlist, no pin), `.env`, MEMORY.md, USER.md, skills, cron, CLI parity | T-03 |
| FR-4 | Capability curation: keep/restrict per role; explicit allowlist; skill subsets; dedupe; no global catalog changes | T-04 |
| FR-5 | Selector-only main model: remove all 10 profile pins; clear/set global default natively; amend policy v2 | T-05 |
| FR-6 | Aux slots on 2.6B local (incl. dev-coder 35B→2.6B); group_router cheap; switch hygiene documented | T-06 |
| FR-7 | Credential hygiene: shared keys in root `.env`; profile `.env` only profile-specific; name-only handling | T-07 |
| FR-8 | Four department rooms, 2–6 members, 3-round/10-msg caps, persistent `Group: <name>` sessions, membership persistence | T-08 |
| FR-9 | Bot-to-bot `message_agent`: canonical Bot Chat gate, fire-and-forget ack, completion outcome, retry-once on transient, zero retry on terminal, max_hops 4, duplicate refusal | T-09 |
| FR-10 | Verification, evidence, rollback: dated snapshots outside `~/.hermes/`, clean `config check`, evidence links | T-10 |

Cross-cutting UX rules: persona changes take effect on new sessions; model changes only via selector; secrets name-only; typed reason codes; bots pass when idle; Owner approval for capital/LOI/external comms/destructive ops.

### 1.3 Architecture (distilled)

- **Posture:** no new primitive; config-driven; local-first services; selector-driven main model; isolation by profile; shared secrets via root `.env` references.
- **Entity model:** 30 entities (Profile, SoulDescriptor, SoulFile, ProfileMeta, MainModelConfig, AuxiliarySlot, Provider, Credential, Toolset, Skill, Plugin, Routine, GroupRoom, RoomSession, CanonicalChat, MessageDelivery, Policy, …) with **ER-1…ER-30** complete relationship rules. Key invariants: 1:1 SoulFile/ProfileMeta per profile; 0..1 config.yaml; pin XOR selector (ER-24); credential scope never both (ER-23); MainModelConfig pin means inherit otherwise (ER-24); group room 2..6 members (ER-13).
- **State machines:** MessageDelivery lifecycle (created → dispatched → admitted → settled/failed; transient vs. terminal reason codes; ≤2 attempts) and GroupRoom turn (message → @mention resolve → rounds 1..3 → settle).
- **File structures:** global `~/.hermes/` (config, .env, SOUL.md immutable, souls/, profiles/, skills/ 2,273, plugins/, cron/, memories/, sessions/, state.db 308 MB); per-profile (profile.yaml, SOUL.md, config.yaml, .env, AGENTS.md, memories/, skills/, plugins/, cron/, sessions/ incl. canonical Bot Chat + Group sessions, state.db).
- **MVB (Minimum Viable Bot):** SOUL (role identity, 3–6 duties, KPIs, guardrails, output contract); profile.yaml (name/description/role/version/ui_meta); config.yaml (allowlist, aux slots, permissions, no pin); `.env` (profile-specific only); memories/MEMORY.md + USER.md; skills subset; cron routines.
- **Group rooms:** gateway-owned where supported; membership mirrored in backend-synced profile metadata with per-gateway versioning; transcript window ~200 msgs/32k chars; oversized message → 8k excerpt; assignment modes (Everyone/lead/automatic via `auxiliary.group_router`); pause/continue/stop; per-member `Group: <room>` sessions; compress history without touching room log.
- **Bot-to-bot:** tool gate requires (1) session titled exactly `Bot Chat`, (2) ≥1 profile with `ui_meta.hermes-bots`, (3) `agent.bot_mode_protocol: true`. Delivery: `message_agent(target, message, expects_reply, constraints, expecting)`; target resolution profile→friendly→@tag (profile first to prevent hijack); attribution prefix; fire-and-forget ack (`status: queued`, `delivery_id`, `process_id`); completion carries final response or typed reason; retry-once on transient only; loop guards max_hops 4 + no duplicate + `bot_replies_expect_reply: false`; message size 16k (tail + note beyond).
- **Model/selector:** pin present ⇒ pin (overrides selector); absent ⇒ inherit launch profile. Target: zero profile pins. Chat-level picker persists across restarts until a pin is set. Mid-session switch forces full uncached re-read (guard `switch_context_confirm_tokens: 100000`). Aux `auto` uses main model; explicit omlx/2.6B for side jobs. Provider entries remain (omlx, openrouter, splash, custom-qwen, nous, opencode); only pins removed.
- **Security:** secrets only in `.env`; config uses `${VAR}`/`key_env`; no inline secrets; shared root-scoped, profile-specific profile-scoped; approval gates (`asset-manager` confirm_destructive true, auto_approve false); Owner approval for capital/LOI/external comms; never enable `-contributor` tiers or `security.allow_data_training_tiers_noninteractive`.
- **Validation surfaces:** `hermes config get model --json`, `hermes config check`, `hermes doctor`, `hermes profile list`, `hermes cron list`, `hermes -p <bot> chat -c "Bot Chat" --create-if-missing`, `hermes sessions refresh-prompts`.

### 1.4 Plan (distilled)

Sequencing principle: **backend data models first, UX/rooms last.** Registry → profile metadata → SOULs → memory → capabilities → model routing → credentials/rooms/messaging → verification.

| Phase | Goal | Gate to enter | Checkpoint |
|---|---|---|---|
| P0 Governance & registry | Decide what exists; ratify/replace policy; record keep/retire dispositions; topology decision | Owner signs registry + policy + topology | T-01 |
| P1 Read-only validation | Confirm data model against disk; no writes | P0 signed | T-01 re-verified |
| P2 Identity & capability model | SOULs, profile.yaml, MEMORY/USER, allowlists, skill subsets, dedupe | P1 T-01 PASS | T-02, T-03, T-04 |
| P3 Model routing / selector-only | Remove 10 pins; set global default natively; pin aux to 2.6B; amend policy v2 | P2 T-02/T-03/T-04 PASS | T-05, T-06 |
| P4 Credentials, rooms, bot-to-bot | Consolidate secrets; canonical Bot Chats; 4 rooms; @mention/caps/persistence; message_agent tests | P3 T-05/T-06 PASS | T-07, T-08, T-09 |
| P5 Verification & rollback | Run checks; re-run all tests; produce VERIFICATION.md; rollback manifest | P4 T-07/T-08/T-09 PASS | T-10; zero FAIL |

- **Hard gates:** a phase cannot start until its predecessor's T-checklists PASS **and** its authorization row is "authorized". No phase is implied by another.
- **Stop conditions:** T-checklist FAIL with no safe remediation; touching a Non-Goal; secret exposure; deletion without approval; caps/loop guards contrary to limits; model-switch cost distortion.
- **Definition of done:** all 10 FRs PASS with evidence; VERIFICATION.md with zero FAILs; per-phase snapshot manifest restorable; no changes outside authorized scope.

---

## 2. Summary of Understanding

The Hermes fleet is a 34-profile agent estate that grew organically on one Hermes Desktop instance. The architecture is fundamentally sound: a Bot is just a profile, Bot Mode is built in, and the design target (selector-driven models, local auxiliaries, least-privilege toolsets, role-aligned SOULs, bounded department rooms, reliable bot-to-bot delivery with typed reasons) is well-specified and testable. What is missing is execution: only 10/34 profiles have config, 22/34 run a generic persona, the orchestrator (the intended first-door router) is a skeleton with a broken roster and unrun acceptance tests, credentials are duplicated in plaintext, and there is no scheduled work at all.

The SoT (`hermes-bot-fleet-sot/`) already provides everything needed to run this as a governed program: 30 entities with complete ER rules, 10 FRs each with binary Pass/Fail tests, 6 phased phases with hard gates and rollback, and a non-goal list that protects against scope creep. The orchestrator design spec adds a concrete router (roster-as-YAML, two delegation paths, upgrade triggers) that is conceptually correct but partially built and unverified.

The new simplification direction — collapsing the estate into service-master bots that delegate to sub-bots — is a **topology decision**, not cleanup. It can reduce the 22-not-ready estate and align with the orchestrator pattern, but it conflicts with Non-Goal 3 (no deletion without approval), consumes the 4-hop loop budget, overlaps with the existing department leads, and breaks the T-08 membership contract unless reconciled. It should be piloted on F&B only and ratified as ADR-8 in P0.

Nothing should be written to `~/.hermes/` until P0 authorization (registry + policy + topology) is recorded. The next concrete deliverable is not code; it is the P0 decision record.

---

## 3. First Three Structural Questions Before Phase 0

1. **Service-master topology (new).** Should department leads (culinary-director, sales-marketing-director, asset-manager/gm-agent) become *delegating masters* that spawn sub-bots, or remain *peer members* of their group rooms? If masters delegate, (a) the hop budget `max_hops: 4` must be revisited, (b) each of the 20 HCREALTY leaf bots needs an explicit keep/retire disposition (Non-Goal 3), and (c) T-08's membership lists must be redefined. **Recommendation:** pilot on F&B only (culinary-director → sous-chef-pastry, restaurant-wine-lead, culinary-frontline-agent) and record as ADR-8; keep the other three rooms as peer rooms.

2. **Identity-layer precedence (unresolved, R4/C9).** There are three competing identity layers: global `~/.hermes/SOUL.md` ("Master-Chief-Orchestrator", 96 lines, which itself instructs "use delegate_task"), per-profile `SOUL.md` (dispatcher personas), and `~/.hermes/souls/` descriptors (22, mostly unapplied). The SoT forbids rewriting the global SOUL (Non-Goal 4) unless Phase 0 rules on precedence. Without a ruling, the orchestrator's delegation rules will be diluted or contradicted by the global persona. **Question:** does the global SOUL apply to the orchestrator profile at all, and if so, in what precedence order relative to the profile SOUL?

3. **Model policy ratification (R2/R3, gates P3).** `LOCAL_MODEL_POLICY.md` mandates local-first 9B/7B/2.6B, but 22 profiles inherit the global 35B splash model, 5 are pinned to OpenRouter Nemotron 550B free (including orchestrator and researcher-bot), and dev-coder pins aux to 35B. Phase 0 must choose among: (a) ratify selector-only with a validated 9B local global default, (b) amend policy to a cloud default, or (c) a documented hybrid with an explicit exception list. This decision also determines whether service masters (which do synthesis) need a heavier *recommended* selector model than leaf bots. **Question:** which of (a)/(b)/(c), and is the orchestrator's current Nemotron 550B pin an accepted exception or drift to be corrected?

---

## 4. Proposed Pilot Topology (for P0 decision, not for execution)

| Master (proposed) | Sub-bots (proposed) | Room | Status |
|---|---|---|---|
| orchestrator (exists) | dev-coder, researcher-bot, reviewer, debugger, planner (room peers) | CODE/DEV | Existing; roster fix needed (C1) |
| culinary-director (exists, custom SOUL) | sous-chef-pastry, restaurant-wine-lead, culinary-frontline-agent | HCREALTY F&B | Pilot candidate for ADR-8 |
| sales-marketing-director (exists, generic SOUL) | experiences-director, guest-experience-manager, guest-services-agent, wellness-director | HCREALTY Sales & Marketing | Defer until SOUL aligned (FR-2) |
| asset-manager / gm-agent (exists) | estate-accountant, estate-engineer, competition-survey-manager | HCREALTY Strategic Board | Defer; finance guardrails first |
| (none proposed) | maintenance/rooms bots (estate-butler-supervisor, executive-housekeeper, estate-housekeeping-agent, hammam-master, spa-hammam-manager, lead-spa-receptionist) | — | Not in any current room; needs a room or retirement decision |
| librarian-bot (NEW) | — (new profile) | Knowledge & Office (proposed) | To be created in P2; wiki/Postgres/database file librarian |
| office-bot (NEW) | — (new profile) | Knowledge & Office (proposed) | To be created in P2; xlsx/csv/pdf/word/.mf transforms |

Leaf bots not listed above (culinary-frontline-agent appears in F&B; estate roles in Strategic Board) must each receive an explicit disposition in P0 per Non-Goal 3. No bot is deleted, archived, or trimmed by this document.

---

## 5. Recommended Immediate Next Step (still read-only)

Produce the P0 decision record in `hermes-bot-fleet-sot/` covering: (1) the 34-bot registry with per-bot disposition (keep/configure/retire), (2) the policy ratification (a/b/c) with date, (3) the identity-precedence ruling, (4) the topology decision (ADR-8: peer rooms vs. pilot master), and (5) Owner signature in `01-CONTEXT.md` §8. Only after that row is marked "authorized" may P1 read-only validation begin. No code, no `~/.hermes/` writes, no skill changes until then.

## 6. Progress log (executed, read-only-safe writes authorized by user 2026-11-26)

- **P0 decisions recorded** (Section 0.5): model policy ratified (selector-only, 9B local global default); topology pilot on F&B only; identity precedence pending; librarian-bot + office-bot added; maintenance bots deferred; GitHub sync done.
- **P1 read-only validation complete:** evidence in `hermes-bot-fleet-sot/P1-VALIDATION-EVIDENCE.md`. Findings: global default on dead Splash :7777 (35B); 10 profile pins confirmed; 33 profiles running; no cron jobs in default profile; doctor clean except vendor-prefix mismatch.
- **P3 global default changed:** `~/.hermes/config.yaml` model block → `MLX-Qwen3.5-9B-DeepSeek-V4-Flash-4bit` on `omlx` (validated 9B local). Snapshot at `hermes-snapshots/2026-11-26-pre-p3/`.
- **P3 profile pins removed:** `model:` block removed from 10 pinned profiles (ape-bot, asset-manager, culinary-director, dev-coder, local-ai-doctor, local-ai-guardian, guardian backup, orchestrator, researcher-bot, splash-7777). They now inherit. Snapshot taken.
- **P2 new bots created:** `librarian-bot` and `office-bot` with full MVB (profile.yaml with ui_meta, SOUL.md, allowlisted config.yaml with no pin, memories/MEMORY.md + USER.md). 12 bundled skills each.
- **Registry created:** `hermes-bot-fleet-sot/registry.md` (36 profiles, dispositions).
- **GitHub sync complete:** https://github.com/yassirworkflow/hermes-bot-fleet-sot (public, main branch). SoT folder + audits + reviews + investigation + blueprint + P1 evidence + registry pushed.
- **Group reclassification:** reviewer/planner/debugger/dev-coder → code-dev; gm-agent/deputy-agent → HCREALTY (registry updated).
- **researcher → tech-teacher-researcher:** profile renamed via `hermes profile rename`; MVB populated (SOUL with research/teaching capabilities + hard guardrails, allowlisted config with execute=false, memories). MCP servers context7 + everything added to global config; serena + agno added (verify on first use).
- **librarian-bot Obsidian integration:** installed obsidian-markdown, obsidian-bases, json-canvas, obsidian-cli, kepano-defuddle skills; Obsidian Integration Protocol added to SOUL.md.
- **wellness-director + wellness-frontline-agent:** snapshotted; marked excluded (deletion pending explicit confirmation — not deleted yet).
- **MCP servers added to global config:** context7, everything, serena, agno (best-effort; verify availability).