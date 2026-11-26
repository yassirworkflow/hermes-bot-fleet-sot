# 03 — ARCHITECTURE

> Tech stack, data models, file structures. **Critical element: State & Schema — complete entity relationship rules.**

---

## 1. Architectural posture

- **No new primitive.** A Bot is a Hermes profile. Bot Mode is a built-in Desktop plugin over profile RPCs; there are no core patches, no new daemons, no new storage.
- **Config-driven, not code-driven.** All changes are file/config edits under `~/.hermes/`, executed only after phase authorization.
- **Local-first services, selector-driven main model.** Shared oMLX `:8888` serves validated local models; main model is chosen by the native selector (no pins); auxiliary slots are pinned small.
- **Isolation by profile.** Config, `.env`, memories, skills, plugins, cron, sessions and `state.db` are per profile; shared secrets live in root `.env` via references.

---

## 2. Tech stack & environment

| Layer | Component | Role |
|---|---|---|
| Desktop | Hermes Desktop (built-in Bot Mode) | Bots pane, group rooms, routines, model selector, profile editing |
| Agent runtime | hermes-agent (CLI/gateway) | `hermes -p <bot> chat`, cron, delegation, profile RPCs |
| Gateway | local gateway (Desktop-owned) | Group-room ownership, delivery ingress, protocol injection |
| Model server | oMLX `:8888` | Shared local inference (9B / 7B / 2.6B validated) |
| Model server | llama-swap `:8085` | researcher-bot accepted exception |
| Search | SearXNG `:8282` | researcher-bot web search |
| Dead service | Splash `:7777` | Known dead; not revived (Non-Goal 6) |
| Cloud providers | OpenRouter, Nous Portal, opencode | Available but not main-model default after P3 |
| Config | `~/.hermes/config.yaml` + profile configs | Precedence: CLI > config > `.env` > defaults |
| Secrets | `~/.hermes/.env` + profile `.env` | `${VAR}` / `key_env` references |
| Storage | profile `state.db`, `sessions/`, `cron/executions.db` | Sessions, messages, routing, routines |
| Docs authority | hermes-agent.nousresearch.com/docs | Bot Mode, Configuration, Configuring Models, Which File Does What |

---

## 3. Entity model (State & Schema)

### 3.1 Entities

| Entity | Key | Attributes | Storage |
|---|---|---|---|
| **Profile (Bot)** | `name` | description, role, version, ui_meta, config, soul, env, memories, skills, plugins, cron, sessions, state | `~/.hermes/profiles/<name>/` |
| **SoulDescriptor** | `name` | description (role intent, KPIs, rules), category | `~/.hermes/souls/<name>-soul.md` |
| **SoulFile** | `profile` | system-prompt body (identity, duties, guardrails, output contract) | `<profile>/SOUL.md` |
| **ProfileMeta** | `name` | name, description, role, version, ui_meta, hidden, section, groups | `<profile>/profile.yaml` |
| **MainModelConfig** | `profile|global` | default, provider, base_url, api_mode, key_env | `config.yaml` `model:` block |
| **AuxiliarySlot** | `profile|global + task` | provider, model, timeout, fallback_chain | `config.yaml` `auxiliary.<task>:` |
| **Provider** | `id` | base_url, api_mode, key_env, discover_models, models[] | `config.yaml` `providers:` |
| **Credential** | `var_name` | scope (root|profile), value (never logged), referenced_by | `.env` files |
| **Toolset** | `profile + name` | enabled/disabled, platform scope | `config.yaml` `toolsets:` / `tools:` |
| **Skill** | `name` | scope (global|profile), category, content | `skills/` dirs |
| **Plugin** | `name` | kind (desktop ESM|agent-side), scope | `~/.hermes/desktop-plugins/`, `plugins/` |
| **Routine** | `[bot:<name>] <label>` | schedule, owner profile, last run | `cron/` + gateway cron |
| **GroupRoom** | `name` | department, members[2..6], owner gateway, transcript, caps, router model | backend-synced metadata + room log |
| **RoomSession** | `room + member` | title `Group: <name>`, messages, compression state | member `sessions/` + `state.db` |
| **CanonicalChat** | `profile` | title exactly `Bot Chat`, pinned, forever-chat | member `sessions/` |
| **MessageDelivery** | `delivery_id` | process_id, sender, target, status, reason_code, attempts, expects_reply | runtime delivery registry + transcript |
| **Policy** | `document` | model assignments, exceptions, classifications | `LOCAL_MODEL_POLICY.md` |

### 3.2 Entity relationship rules (complete)

| # | Rule | Cardinality / constraint |
|---|---|---|
| ER-1 | Profile **has exactly one** SoulFile | 1:1; file must exist (Hermes seeds starter if missing) |
| ER-2 | SoulDescriptor **maps to at most one** applied SoulFile | 1:0..1; descriptors may exist unapplied (current state: 22 descriptors, 1 applied) |
| ER-3 | Profile **has exactly one** ProfileMeta (`profile.yaml`) | 1:1; two profiles currently violate this (`local-ai-doctor`, `splash-7777`) — FR-1 fixes |
| ER-4 | Profile **has zero or one** `config.yaml` | 0..1; absence = inherit global settings |
| ER-5 | Profile **has one** `.env` (may be empty) | 1:1; shared keys referenced, not duplicated |
| ER-6 | Profile **has one** `memories/` with MEMORY.md + USER.md | 1:2 minimum; USER.md currently missing in all — FR-3 fixes |
| ER-7 | Profile **has N** Skills (profile-scoped copies) | 1:N; observed 12–45, backup 2,251 |
| ER-8 | Global Skill **can be present in N** profiles | N:M via copies; global catalog unaffected by profile deletion |
| ER-9 | Profile **has 0..N** Plugins | 1:N; agent-side plugins shared at gateway, enabled per profile |
| ER-10 | Profile **has 0..N** Routines | 1:N; namespaced `[bot:<name>]`; appear in `hermes cron list` |
| ER-11 | Profile **has 1..N** sessions, including 1 CanonicalChat | 1:N; canonical titled exactly `Bot Chat` |
| ER-12 | CanonicalChat **is the only** session that receives `message_agent` + bot protocol | gate: title `Bot Chat` AND ≥1 ProfileMeta has `ui_meta.hermes-bots` AND `agent.bot_mode_protocol: true` |
| ER-13 | GroupRoom **has 2..6** Profiles | 1:N bounded; membership in backend-synced metadata |
| ER-14 | Profile **belongs to 0..N** GroupRooms | N:M; one DM row per bot plus one row per room |
| ER-15 | GroupRoom **has one** RoomSession **per member** | 1:N; title `Group: <room>`, persists across closes |
| ER-16 | GroupRoom transcript **is bounded** | ≤200 messages / ≤32,000 chars; oversized single message → 8,000-char excerpt |
| ER-17 | GroupRoom turn **capped** | ≤3 serial rounds; ≤10 messages per send; gateway-owned pause or classic stop |
| ER-18 | GroupRoom **may use one** AuxiliarySlot `group_router` | 0..1; cheap model for automatic assignment |
| ER-19 | MessageDelivery **has exactly one** sender and one target Profile | N:1 + N:1 |
| ER-20 | MessageDelivery **has at most one** completion result | 1:0..1; carries final response or typed reason |
| ER-21 | MessageDelivery **attempts ≤ 2** | initial + at most one retry; retry only on transient reason codes |
| ER-22 | MessageDelivery **loop guard** | ≤`max_hops` (4) per exchange; duplicate (sender,target,message) refused |
| ER-23 | Credential **scope is root or profile, never both** | shared keys root; profile-specific keys profile; no inline config secrets |
| ER-24 | MainModelConfig **pin XOR selector** | If `model.default`/`provider` set in a profile → pin; else inherit global. Target state: **no profile pins** |
| ER-25 | AuxiliarySlot `provider: auto` **uses main model** | With no fallback declared and main unavailable → task skipped with warning, not silently billed elsewhere |
| ER-26 | Policy **classifies every Profile** | CANONICAL / INTENTIONAL_MINIMAL / ACCEPTED_EXCEPTION / DRIFT / RISK / REVIEW |
| ER-27 | Config precedence | CLI args > `config.yaml` > `.env` > built-in defaults; profile `.env` wins for `${VAR}` in multiplexed gateways |
| ER-28 | SOUL + MEMORY/USER **are prompt inputs**, not live state | SOUL = slot #1 at session start; MEMORY/USER = frozen snapshot; edits need new session or `refresh-prompts` |
| ER-29 | Desktop plugins **are ESM in `~/.hermes/desktop-plugins/`** | UI extensions only; agent-side plugins are Python packages shared at gateway |
| ER-30 | Every Bot **is CLI-addressable** | `hermes -p <name> chat`; `hermes profile create <name>` creates isolated profile |

### 3.3 State machines

**MessageDelivery lifecycle:**

```
created → dispatched(queued + delivery_id + process_id)
        → admitted(target picks up when it next runs)
        → settled(completion notification carries reply)
        → failed(typed reason_code) → [transient? retry-once : report]
```

Transient (retry allowed): `runtime_offline`, `delivery_timeout`, `provider_rate_limit`, `provider_server_error`, `context_overflow`.
Terminal (no retry): `provider_auth_or_access`, `provider_quota_limit`, `missing_config`, `model_unavailable`, `target_busy`, `queued_expired`, `unknown`.

**GroupRoom turn:**

```
message → resolve @mentions (or everyone) → round 1..3
        → member replies/pass/silence → count messages (≤10)
        → full silent round OR cap → settle (gateway-owned: pause+continue/stop)
```

---

## 4. File structures

### 4.1 Global (`~/.hermes/`)

```
~/.hermes/
├── config.yaml                 # global settings; main model pin to be cleared in P3
├── .env                        # ROOT shared secrets (consolidated in P4)
├── auth.json                   # OAuth credentials
├── SOUL.md                     # global identity — DO NOT MODIFY (Non-Goal 4)
├── souls/                      # 22 role descriptors (source for profile SOULs)
│   └── <bot>-soul.md
├── profiles/                   # 34 bot profiles
├── skills/                     # 2,273 global skills (DO NOT MODIFY — Non-Goal 5)
├── plugins/                    # agent-side plugins (shared at gateway)
├── desktop-plugins/            # ESM UI extensions (DO NOT ADD — Non-Goal 11)
├── cron/                       # global cron runtime
├── memories/                   # global MEMORY.md / USER.md
├── sessions/                   # gateway sessions
└── state.db                    # sessions, messages, routing
```

### 4.2 Per profile (`~/.hermes/profiles/<name>/`)

```
<name>/
├── profile.yaml                # REQUIRED after FR-1: name, description, role, version, ui_meta
├── SOUL.md                     # REQUIRED: role identity + guardrails
├── config.yaml                 # OPTIONAL: tool allowlist, aux slots; NO main-model pin after P3
├── .env                        # profile-specific secrets only
├── AGENTS.md                   # OPTIONAL: project-specific instructions
├── memories/
│   ├── MEMORY.md               # agent notes
│   └── USER.md                 # owner preferences
├── skills/                     # role-relevant subset (12–45)
├── plugins/                    # profile-specific (optional)
├── cron/                       # routines + runtime artifacts
├── sessions/                   # includes canonical "Bot Chat" + "Group: <room>" sessions
├── state.db                    # profile state
└── assets/, cache/, logs/, ... # runtime
```

### 4.3 Minimum Viable Bot Configuration (MVB) — the required baseline

| File | Minimum content | Enhancement in this program |
|---|---|---|
| `SOUL.md` | Role identity, 3–6 duties, KPIs, guardrails, output contract | Rewrite from `souls/` descriptor; strip generic persona |
| `profile.yaml` | `name`, `description`, `role`, `version`, `ui_meta: {hermes-bots: {}}` | Create/fix for all 34 |
| `config.yaml` | `tools:` allowlist, `auxiliary.*` slots, `permissions` for sensitive roles | Add allowlist; remove model pin; set aux to 2.6B |
| `.env` | Profile-specific vars only | Remove duplicated shared keys; reference root via `key_env` |
| `memories/MEMORY.md` | Environment facts, workspace paths | Replace 38-byte stub |
| `memories/USER.md` | Owner preferences (e.g. non-technical explanations) | Create everywhere |
| `AGENTS.md` | Project rules if the bot works in a repo | Add only where needed |
| `skills/` | Role-relevant subset | Curate per FR-4 |
| `cron/` | `[bot:<name>]` routines | Verify, don't invent |

---

## 5. Group chat architecture (FR-8)

- Rooms are **gateway-owned** where the gateway advertises group-chat methods; they keep running with Desktop closed. Classic rooms (older) stop when Desktop closes — not created here.
- Membership and room identity are mirrored into the **shared profile metadata of every connected gateway** with per-gateway versioning; a rename/disband is durable.
- Each member turn carries messages since its own last turn, up to ~200 messages / 32,000 chars; single oversized message → marked 8,000-char excerpt.
- Assignment when nobody is @mentioned: room setting (Everyone default / lead / automatic). Automatic uses `auxiliary.group_router` cheap model over room purpose, member roles (profile descriptions) and last messages.
- Hard caps: 10 messages per send, 3 rounds; gateway-owned rooms pause with Continue (≤5 times) / Stop.
- Stop directives only adjacent to @mention; code/quotes ignored; `Detect stop directives` toggle available.
- Per-member `Group: <room>` sessions are ordinary Hermes sessions — can be resumed from CLI or posted to by routines; room re-posts external writes once.
- `Compress history` per member folds large hidden room sessions without touching the room log.

**Room definitions (memership ≤6):**

| Room | Department | Members |
|---|---|---|
| HCREALTY F&B | f&b | culinary-director, sous-chef-pastry, restaurant-wine-lead, culinary-frontline-agent |
| HCREALTY Sales & Marketing | sales&marketing | sales-marketing-director, experiences-director, guest-experience-manager, guest-services-agent, wellness-director |
| HCREALTY Strategic Board | strategic-board | asset-manager, gm-agent, estate-accountant, estate-engineer, competition-survey-manager |
| CODE/DEV | code-dev | orchestrator, dev-coder, reviewer, debugger, planner, researcher |

---

## 6. Bot-to-bot architecture (FR-9)

**Tool availability gate (all three must hold):**
1. Session titled exactly `Bot Chat`.
2. ≥1 profile with `ui_meta: { hermes-bots: {} }` (currently `orchestrator` satisfies this).
3. `agent.bot_mode_protocol: true` (default).

**Delivery mechanics:**
- `message_agent(target, message, expects_reply, constraints, expecting)`.
- Target resolution: profile name → friendly name → @tag; profile name matched first (prevents hijack); ambiguous friendly name refused with roster.
- Attribution prefix added automatically: `Message from 🤖 <friendly name> (@<handle>):`.
- Fire-and-forget ack: `status: queued`, `delivery_id`, `process_id` = handed to background process, not delivered.
- Completion notification carries recipient's final response as an outcome sentence; silence → empty reply.
- Cross-connection: Desktop relays over connection sockets; only text crosses machines; reply returns as background completion.
- No live interrupt of a mid-conversation bot (future work).

**Typed reason codes (machine-readable, end-to-end):**

`provider_auth_or_access` · `provider_quota_limit` · `provider_rate_limit` · `provider_server_error` · `context_overflow` · `missing_config` · `model_unavailable` · `runtime_offline` · `queued_expired` · `delivery_timeout` · `target_busy` · `unknown`

**Retry policy:**

| Class | Codes | Attempts |
|---|---|---|
| Transient | runtime_offline, delivery_timeout, provider_rate_limit, provider_server_error, context_overflow | 1 + 1 retry, same session |
| Terminal | provider_auth_or_access, provider_quota_limit, missing_config, model_unavailable, target_busy, queued_expired, unknown | 1, report immediately |

**Loop guards:** `gateway.bot_loop_guard.max_hops: 4`; `bot_replies_expect_reply: false`; duplicate send refused; bot-to-bot reply becomes notify-only.

---

## 7. Model & selector architecture (FR-5/6)

- **Pin vs inherit invariant:** profile `model.default`/`provider` present ⇒ pin (overrides selector); absent ⇒ inherit launch profile. Target: zero profile pins.
- **Selector scope:** chat-level picker sticks across app restarts until a profile model pin is set; hot-swap via `/model`; config changes apply to new sessions only.
- **Cache cost:** mid-session switch rewrites `Model:`/`Provider:` lines in system prompt → one uncached full re-read; `switch_context_confirm_tokens: 100000` default guard.
- **Auxiliary routing:** `auto` uses main model; explicit `omlx`/2.6B for side jobs; `group_router` for room assignment.
- **Provider entries remain** (omlx, openrouter, splash, custom-qwen, nous, opencode) for selector choice; only pins removed.

---

## 8. Security architecture (FR-7)

- Secrets only in `.env`; config stores `${VAR}` / `key_env`; Hermes writes credentials this way for pickers.
- Shared keys root-scoped; profile-specific keys profile-scoped; no value in config or reports.
- Approval gates: `asset-manager` `confirm_destructive: true`, `auto_approve: false`; Owner approval for capital/LOI/external comms; group rooms surface approvals as "needs you".
- Tool isolation per profile; least privilege per FR-4.
- Never enable `-contributor` tiers or `security.allow_data_training_tiers_noninteractive`.

---

## 9. Validation surfaces

| Command | Purpose | Phase |
|---|---|---|
| `hermes config get model --json` | Effective model config | P1, P3 |
| `hermes config check` | Missing/stale options | P1–P5 |
| `hermes doctor` | Health warnings | P5 |
| `hermes profile list` | Roster parity | P1 |
| `hermes cron list` | Routine inventory | P1 |
| `hermes -p <bot> chat -c "Bot Chat" --create-if-missing` | Canonical chat | P4 |
| `hermes sessions refresh-prompts` | Adopt SOUL/memory edits in long sessions | P2 |

---

## 10. Architectural decisions (ADRs)

| ADR | Decision | Rationale | Owner |
|---|---|---|---|
| ADR-1 | Bot = profile; use built-in Bot Mode | No new primitive; archived plugin repo ignored | P0 |
| ADR-2 | Selector-only main model; remove all pins | User requirement; pins override selector | P0/P3 |
| ADR-3 | Auxiliary slots pinned to 2.6B local | Cost/quality per policy §4 | P0/P3 |
| ADR-4 | SOULs sourced from `souls/` descriptors | Role intent already authored | P2 |
| ADR-5 | Shared secrets root-scoped | Policy §6; reduce blast radius | P4 |
| ADR-6 | Four named rooms with bounded membership | Department coordination requirement | P4 |
| ADR-7 | No profile deletion without explicit approval | Non-Goal 3 | P0 |
