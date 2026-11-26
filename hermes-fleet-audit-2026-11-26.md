# Hermes Desktop Bot Fleet — Audit Report

**Date:** 2026-11-26 (audit run)
**Scope:** Read-only audit of `~/.hermes/profiles/` (34 profiles) + global `~/.hermes/`. No files modified.
**Source evidence:** `hermes profile list`, `hermes doctor`, per-profile `config.yaml`/`profile.yaml`/`SOUL.md`/`.env`/`skills/`/`plugins/`/`memories/`, `~/.hermes/souls/` (22 descriptors), `LOCAL_MODEL_POLICY.md`.

## 1. Executive summary

- **34 profile directories** (doctor reports 33; `.deleted` empty, `active_profile` is a pointer file). All 33 registered profiles report gateway **running**.
- Only **10/34 have `config.yaml`**; 24 inherit the global model (`mlx-community/Qwen3.6-35B-A3B-4bit` on `splash` provider) — doctor flags the global pin as vendor-prefixed mismatch.
- **Model pins present on 12 profiles** (10 via config.yaml; 3 via profile.yaml: asset-manager, competition-survey-manager, sales-marketing-director — asset-manager overlaps). 22 profiles have no pin → inherit global (cloud/local drift per LOCAL_MODEL_POLICY).
- **SOUL: 9 custom, 25 generic default** (663-byte Nous Research boilerplate). 22 role descriptors exist in `~/.hermes/souls/` but only asset-manager (and a few others) actually use them → persona/role gap for ~18 estate bots.
- **Tools allowlist defined for only 3 profiles** (asset-manager, competition-survey-manager, sales-marketing-director). All others rely on broad `platform_toolsets`.
- **MCP servers declared for only 3 profiles** (same three). Knowledge base + memory_files declared for only those 3.
- **Memory files populated for only 4 profiles** (orchestrator, researcher-bot, local-ai-guardian, local-ai-guardian backup). Most have empty memory.
- **Credentials sprawl:** 6 profiles carry `.env` with API keys; local-ai-doctor (17 keys) and orchestrator (16 keys) duplicate shared keys (policy §6 violation).
- **Cron:** zero scheduled jobs across all profiles (only runtime ticker artifacts).
- **Skills:** 25 profiles share identical 12 base skill categories; dev-coder/local-ai-doctor/orchestrator/researcher-bot have role-specific extras; local-ai-guardian has 45 (+ duplicates); backup has 2,263 (outlier).
- **Readiness:** 8 🟢 ready, 4 🟡 partial, 22 🔴 not ready (mostly generic-SOUL estate bots missing config).

## 2. Global baseline (shared by all unpinned profiles)

| Item | Value |
|---|---|
| Global model default | `mlx-community/Qwen3.6-35B-A3B-4bit` (provider `splash`, base_url 127.0.0.1:7777) |
| Global model provider | `splash` (⚠ doctor: vendor-prefixed slug mismatch) |
| OpenRouter default_provider | `omlx` (fallback_providers: `omlx`) |
| Auxiliary web_extract | `LiquidAI--LFM2.5-2.6B-MLX-4bit` via omlx |
| Other auxiliary slots | unset (auto) |
| Memory provider | honcho (memory_enabled, user_profile_enabled) |
| Platform toolsets | a2a, browser, clarify, code_execution, computer_use, connections, cronjob, delegation, file, kanban, memory, omh, session_search, skills, terminal, todo, vision, web |
| Tools | tool_search auto (threshold 10%) |
| Skills disabled (global) | 1,633 |
| Global SOUL | Custom Master-Chief-Orchestrator (96 lines) |
| Global USER.md | present (965 bytes) |
| State DB | 308 MB (WAL) |
| Secrets | 19 keys in root `.env` |
| Auth providers | Nous Portal / OpenAI Codex / MiniMax: NOT logged in |
| Tool availability gaps | discord (no token), image_gen (no provider), x_search (no XAI key), a2a/browser-cdp/connections/feishu/google_meet/homeassistant (system deps) |

## 3. Fleet overview table

| Profile | Group | cfg | Model pin | Provider | SOUL | Skills | Plugins | Tools/Perms | MCP | Mem | Env | Soul-lib | Readiness |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| ape-bot | AI-Infra | Y | MLX-Qwen3.5-9B-DeepSeek-V4-Flash-4bit | omlx | custom | 18 | — | default | — | — | — | — | 🟢 Ready |
| asset-manager | HCREALTY | Y | nvidia/nemotron-3-ultra-550b-a55b:free | openrouter | custom | 24 | — | allowlist | yes | — | — | Y | 🟢 Ready |
| competition-survey-manager | HCREALTY | — | profile | profile | custom | 15 | — | allowlist | yes | — | — | Y | 🟢 Ready |
| culinary-director | HCREALTY | Y | qwen3.6-plus-free | opencode | generic-default | 15 | — | default | — | — | 2 | Y | 🟡 Partial |
| culinary-frontline-agent | HCREALTY | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | Y | 🔴 Not ready |
| debugger | Specialist | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | — | 🔴 Not ready |
| deputy-agent | Specialist | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | Y | 🔴 Not ready |
| dev-coder | AI-Infra | Y | Qwen2.5-Coder-7B-Instruct-4bit | omlx | custom | 28 | rtk-rewrite-docker,snyk,openspec | default | — | — | — | — | 🟢 Ready |
| estate-accountant | HCREALTY | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | Y | 🔴 Not ready |
| estate-butler-supervisor | HCREALTY | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | Y | 🔴 Not ready |
| estate-engineer | HCREALTY | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | Y | 🔴 Not ready |
| estate-housekeeping-agent | HCREALTY | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | Y | 🔴 Not ready |
| executive-housekeeper | HCREALTY | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | Y | 🔴 Not ready |
| experiences-director | HCREALTY | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | Y | 🔴 Not ready |
| gm-agent | Specialist | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | Y | 🔴 Not ready |
| guest-experience-manager | HCREALTY | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | Y | 🔴 Not ready |
| guest-services-agent | HCREALTY | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | Y | 🔴 Not ready |
| hammam-master | HCREALTY | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | Y | 🔴 Not ready |
| lead-spa-receptionist | HCREALTY | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | Y | 🔴 Not ready |
| local-ai-doctor | AI-Infra | Y | MLX-Qwen3.5-9B-DeepSeek-V4-Flash-4bit | omlx | custom | 28 | — | default | — | — | 17 | — | 🟡 Partial |
| local-ai-guardian | AI-Infra | Y | MLX-Qwen3.5-9B-DeepSeek-V4-Flash-4bit | omlx | custom | 57 | langfuse,hermes-session-reset-policy,ponytail,honcho | default | — | yes | 12 | — | 🟢 Ready |
| local-ai-guardian.backup.20261103-142924 | AI-Infra | Y | MLX-Qwen3.5-9B-DeepSeek-V4-Flash-4bit | omlx | custom | 2263 | langfuse,omh,hermes-session-reset-policy,homeassistant,rtk-rewrite,ponytail,honcho,superpowers | default | — | yes | 11 | — | 🟢 Ready |
| orchestrator | AI-Infra | Y | nvidia/nemotron-3-ultra-550b-a55b:free | openrouter | custom | 25 | — | default | — | yes | 16 | — | 🟢 Ready |
| planner | Specialist | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | — | 🔴 Not ready |
| researcher | Specialist | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | — | 🔴 Not ready |
| researcher-bot | AI-Infra | Y | nvidia/nemotron-3-ultra-550b-a55b:free | openrouter | custom | 29 | orca-status | default | — | yes | 8 | — | 🟢 Ready |
| restaurant-wine-lead | HCREALTY | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | Y | 🔴 Not ready |
| reviewer | Specialist | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | — | 🔴 Not ready |
| sales-marketing-director | HCREALTY | — | profile | profile | generic-default | 15 | — | allowlist | yes | — | — | Y | 🟡 Partial |
| sous-chef-pastry | HCREALTY | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | Y | 🔴 Not ready |
| spa-hammam-manager | HCREALTY | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | Y | 🔴 Not ready |
| splash-7777 | AI-Infra | Y | mlx-community--Qwen3.6-35B-A3B-4bit | omlx | generic-default | 16 | — | default | — | — | — | — | 🟡 Partial |
| wellness-director | HCREALTY | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | Y | 🔴 Not ready |
| wellness-frontline-agent | HCREALTY | — | inherit | inherit | generic-default | 15 | — | default | — | — | — | Y | 🔴 Not ready |

Legend: cfg = has profile config.yaml; SOUL = custom vs generic-default; Skills = entries in profile/skills/; Plugins = dirs in profile/plugins/; Tools/Perms = profile-level allowlist vs default broad toolsets; Mem = memory files populated; Env = number of keys in profile `.env`; Soul-lib = matching descriptor in `~/.hermes/souls/`.

## 4. Findings by risk (from investigation, re-verified)

| ID | Risk | Evidence (this audit) | Severity |
|---|---|---|---|
| R1 | Ghost/missing profiles | `researcher` has no config.yaml AND no profile.yaml; `local-ai-doctor` and `splash-7777` have config.yaml but no profile.yaml; `.deleted/` empty | High |
| R2 | Policy drift (cloud vs local) | 22 unpinned profiles inherit global 35B splash; 5 pinned to OpenRouter Nemotron 550B free; LOCAL_MODEL_POLICY contradicted | High |
| R3 | Native selector blocked by pins | model.default/provider set in 10 configs + global; profile.yaml pins in 3 | High |
| R4 | Persona/role mismatch | 25 generic SOUL; 22 souls/ descriptors unused except asset-manager; financial/ops approval gates absent outside asset-manager | High |
| R5 | Credential sprawl | 6 profile `.env` with duplicated plaintext keys (doctor/ orchestrator 16-17 keys each) | High |
| R6 | Context bloat | guardian backup 2,263 skills; global 2,273; duplicate skills (systematic-debugging x3) | Medium-High |
| R7 | Over-broad toolsets | default platform_toolsets include browser/computer_use/image_gen/delegation/cronjob for all; only 3 allowlists | Medium |
| R8 | Empty memory | only 4 profiles have memory files; orchestrator root MEMORY.md = 38 bytes | Low-Medium |
| R9 | Cron unverified | zero scheduled jobs; only runtime tickers | Medium |
| R10 | Stale backup profile | local-ai-guardian.backup.20261103-142924 (2,263 skills, _config_version 52) | Low-Medium |

## 5. Per-group notes

### 5.1 AI-Infra / Core (8)
- **orchestrator**: custom dispatcher SOUL (2.5KB), 25 skills incl. orchestrator/rtk, 0 plugins, memory populated (memories/MEMORY.md 1.9KB + USER.md 0.2KB), pinned OpenRouter Nemotron. Role: hub/router, no domain work. Production-ready but memory stub at root (38B) is misleading (real memory in memories/).
- **researcher-bot**: custom research persona (8.3KB), 29 skills (evidence-ledger, search-strategy, etc.), 1 plugin (orca-status), pinned OpenRouter Nemotron, memory USER.md only (11 bytes). Env has search keys (SEARXNG, BRAVE, WEBSURFX).
- **dev-coder**: custom coding SOUL (10KB), 28 skills, 3 plugins (rtk-rewrite-docker, snyk, openspec), pinned local 7B coder, aux all pinned to 35B (policy drift: should be 2.6B). platform_toolsets default.
- **ape-bot**: custom APE interface SOUL (2.9KB), 18 skills, 0 plugins, pinned 9B local. Env empty (inherits root).
- **local-ai-doctor**: custom diagnostician SOUL (2.8KB), 28 skills (local-ai-diagnostics, mlops, ollama, etc.), 0 plugins (config enables 2 but none installed), pinned 9B local. Env has 17 keys (sprawl). No profile.yaml.
- **local-ai-guardian**: custom guardian SOUL (0.5KB), 57 skills, 4 plugins (langfuse, honcho, ponytail, session-reset-policy), memory populated (2KB). Pinned 9B local. Broadest config (full toolset).
- **local-ai-guardian.backup.20261103-142924**: duplicate of guardian with 2,263 skills, 8 plugins, _config_version 52 (stale). Should be archived.
- **splash-7777**: generic SOUL, 16 skills, no profile.yaml, pinned splash 35B. No tools/env. Doctor flags global vendor-prefix mismatch partly due to this provider.

### 5.2 HCREALTY Estate (20)
- Only **3** have rich profile.yaml (asset-manager, competition-survey-manager, sales-marketing-director) with model pin (OpenRouter Nemotron), tools allowlist (11 tools), permissions (read/write/execute/network, delete=ask), settings (auto_approve false), 20 skills, 5 MCP servers (filesystem, github, postgres, web-search, market-data), knowledge_base path, memory_files (PIPELINE/ASSETS/COMP-SET).
- **asset-manager** is the only one with a custom SOUL (2.8KB) actually written; competition-survey-manager and sales-marketing-director have generic 663B SOUL despite rich profile.yaml → persona gap.
- The other **17 estate bots** have minimal profile.yaml (name/description/role/version only), generic SOUL, 15 skills (12 base categories), no config.yaml, no tools allowlist, no MCP, no KB, no memory, no env. They inherit global model + broad toolsets.
- **22 role descriptors** in `~/.hermes/souls/` (asset-manager plus 21 estate roles) are skill-style metadata (name/description/category) encoding KPIs/guardrails, but only asset-manager rules are in its SOUL.md.
- Roles present as descriptors: asset-manager, competition-survey-manager, culinary-director, culinary-frontline-agent, deputy-agent, estate-accountant, estate-butler-supervisor, estate-engineer, estate-housekeeping-agent, executive-housekeeper, experiences-director, gm-agent, guest-experience-manager, guest-services-agent, hammam-master, lead-spa-receptionist, restaurant-wine-lead, sales-marketing-director, sous-chef-pastry, spa-hammam-manager, wellness-director, wellness-frontline-agent.
- Missing descriptors (no souls/ file): debugger, planner, reviewer, researcher, researcher-bot, dev-coder, local-ai-doctor, local-ai-guardian, orchestrator, ape-bot, splash-7777.

### 5.3 Specialists (6)
- debugger, planner, reviewer, researcher: minimal profile.yaml (name/description/role/version), generic SOUL, 15 base skills, no config, no env. Roles: debugger, planner, reviewer, researcher (note: researcher has NO profile.yaml at all — ghost).
- deputy-agent: minimal profile, generic SOUL, role=general (despite description Estate Manager/COO).
- gm-agent: minimal profile, generic SOUL, role=orchestrator (delegation intent) but no tools/delegation config.

## 6. Status / health

- **Gateway:** all 33 registered profiles running (per doctor).
- **Doctor issues:** 1 (global model vendor-prefix mismatch: `mlx-community/Qwen3.6-35B-A3B-4bit` with provider `splash`). 24 profiles flagged missing config (inherit global).
- **Auth:** Nous Portal, OpenAI Codex, MiniMax not logged in (optional).
- **Tool availability gaps:** discord, image_gen, x_search, a2a, browser-cdp, connections, feishu, google_meet, homeassistant (system deps or missing keys).
- **State DB:** 308 MB (potential bloat; WAL mode).
- **Cron:** zero jobs; only runtime tickers.
- **Memory:** honcho provider active; only 4 profiles populated.

## 7. Production-readiness verdict (per profile)

| Profile | Verdict | Key blockers |
|---|---|---|
| ape-bot | 🟢 Ready | no tool allowlist (broad default toolsets); empty memory |
| asset-manager | 🟢 Ready | empty memory |
| competition-survey-manager | 🟢 Ready | no config.yaml (inherits global); empty memory |
| culinary-director | 🟡 Partial | generic SOUL (no role persona/guardrails); no tool allowlist (broad default toolsets); empty memory |
| culinary-frontline-agent | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory |
| debugger | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory |
| deputy-agent | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory |
| dev-coder | 🟢 Ready | no tool allowlist (broad default toolsets); empty memory |
| estate-accountant | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory |
| estate-butler-supervisor | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory |
| estate-engineer | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory |
| estate-housekeeping-agent | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory |
| executive-housekeeper | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory |
| experiences-director | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory |
| gm-agent | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory |
| guest-experience-manager | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory |
| guest-services-agent | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory |
| hammam-master | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory |
| lead-spa-receptionist | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory |
| local-ai-doctor | 🟡 Partial | no tool allowlist (broad default toolsets); empty memory; credential sprawl (.env); missing profile.yaml |
| local-ai-guardian | 🟢 Ready | no tool allowlist (broad default toolsets); credential sprawl (.env) |
| local-ai-guardian.backup.20261103-142924 | 🟢 Ready | no tool allowlist (broad default toolsets); credential sprawl (.env); stale backup (2,263 skills) |
| orchestrator | 🟢 Ready | no tool allowlist (broad default toolsets); credential sprawl (.env) |
| planner | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory |
| researcher | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory; missing profile.yaml |
| researcher-bot | 🟢 Ready | no tool allowlist (broad default toolsets); credential sprawl (.env) |
| restaurant-wine-lead | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory |
| reviewer | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory |
| sales-marketing-director | 🟡 Partial | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); empty memory |
| sous-chef-pastry | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory |
| spa-hammam-manager | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory |
| splash-7777 | 🟡 Partial | generic SOUL (no role persona/guardrails); no tool allowlist (broad default toolsets); empty memory; missing profile.yaml |
| wellness-director | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory |
| wellness-frontline-agent | 🔴 Not ready | no config.yaml (inherits global); generic SOUL (no role persona/guardrails); no model pin; no tool allowlist (broad default toolsets); empty memory |

## 8. Recommendations (authorization-gated; no changes made)

1. Adopt the discovered 34-profile registry as source of truth; resolve `researcher` (retire or configure), `splash-7777`, and the guardian backup.
2. Give every bot a complete `profile.yaml` (name/description/role/version) and a role-specific SOUL.md sourced from `~/.hermes/souls/<bot>-soul.md` (write the KPIs/guardrails into SOUL.md, not just metadata).
3. Ratify or replace `LOCAL_MODEL_POLICY.md`; currently hybrid cloud/local with drift. Decide native-selector-only end-state (unset all model pins) vs pinned policy.
4. Define per-profile tool allowlists (only 3 have them); remove browser/computer_use/image_gen/cronjob/delegation from leaf/finance bots.
5. Consolidate duplicated API keys from profile `.env` files into root `.env` (policy section 6); remove plaintext sprawl in local-ai-doctor and orchestrator.
6. Archive/remove the 2,263-skill guardian backup; dedupe skills (systematic-debugging x3).
7. Populate memory (USER.md at minimum) for orchestrator and researcher-bot; keep MEMORY.md terse.
8. Run `hermes config check`, `hermes doctor`, and per-bot smoke tests before any production promotion.

---
End of audit. Read-only; no files under `~/.hermes/` modified.