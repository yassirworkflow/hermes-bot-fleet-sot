# Hermes Bot Fleet — Authoritative Registry

**Version:** 1.1 (P0/P2 update, 2026-11-26)
**Status:** Adopted as source of truth. Supersedes the supplied inventory.
**Scope:** 36 profile directories (34 discovered + 2 created: librarian-bot, office-bot). `profiles/.deleted/` is empty and is not a profile. Backup profile suffix `.backup.20261103-142924` is NOT counted as an active bot.

## Dispositions (P0 decisions)

- `researcher` (ghost, no config/profile): **configure** — keep, give full MVB in P2 follow-up.
- `splash-7777`: keep pending; model pin removed (P3); generic SOUL.
- `local-ai-guardian.backup.20261103-142924`: keep pending Owner approval; do not trim silently (Non-Goal 3).
- Maintenance/rooms bots (estate-butler-supervisor, executive-housekeeper, estate-housekeeping-agent, hammam-master, spa-hammam-manager, lead-spa-receptionist): **deferred to later batch** after hermes-core-bot-fleet.
- `librarian-bot`, `office-bot`: **created** in P2 (this update) with MVB.

## Registry table

| # | Profile | Group | cfg | Model pin (post-P3) | SOUL | Skills | Tools | Readiness | Disposition |
|---|---|---|---|---|---|---|---|---|---|
| 1 | ape-bot | AI-Infra | Y | none (inherited) | custom | 18 | default (to be allowlisted) | 🟢 | keep |
| 2 | asset-manager | HCREALTY | Y | none | custom | 24 | allowlist | 🟢 | keep |
| 3 | competition-survey-manager | HCREALTY | — | none | generic (to fix) | 15 | allowlist | 🟢 | keep |
| 4 | culinary-director | HCREALTY | Y | none | generic (to fix) | 15 | default | 🟡 | keep (F&B master pilot) |
| 5 | culinary-frontline-agent | HCREALTY | — | none | generic (to fix) | 15 | default | 🔴 | keep (F&B sub-bot) |
| 6 | debugger | code-dev | — | none | generic (to fix) | 15 | default | 🔴 | keep |
| 7 | deputy-agent | HCREALTY | — | none | generic (to fix) | 15 | default | 🔴 | keep |
| 8 | dev-coder | code-dev | Y | none | custom | 28 | default (to be allowlisted) | 🟢 | keep |
| 9 | estate-accountant | HCREALTY | — | none | generic (to fix) | 15 | default | 🔴 | keep (Strategic Board) |
| 10 | estate-butler-supervisor | HCREALTY | — | none | generic (to fix) | 15 | default | 🔴 | deferred (later batch) |
| 11 | estate-engineer | HCREALTY | — | none | generic (to fix) | 15 | default | 🔴 | keep (Strategic Board) |
| 12 | estate-housekeeping-agent | HCREALTY | — | none | generic (to fix) | 15 | default | 🔴 | deferred (later batch) |
| 13 | executive-housekeeper | HCREALTY | — | none | generic (to fix) | 15 | default | 🔴 | deferred (later batch) |
| 14 | experiences-director | HCREALTY | — | none | generic (to fix) | 15 | default | 🔴 | keep (Sales & Mktg) |
| 15 | gm-agent | HCREALTY | — | none | generic (to fix) | 15 | default | 🔴 | keep (Strategic Board) |
| 16 | guest-experience-manager | HCREALTY | — | none | generic (to fix) | 15 | default | 🔴 | keep (Sales & Mktg) |
| 17 | guest-services-agent | HCREALTY | — | none | generic (to fix) | 15 | default | 🔴 | keep (Sales & Mktg) |
| 18 | hammam-master | HCREALTY | — | none | generic (to fix) | 15 | default | 🔴 | deferred (later batch) |
| 19 | lead-spa-receptionist | HCREALTY | — | none | generic (to fix) | 15 | default | 🔴 | deferred (later batch) |
| 20 | librarian-bot | Knowledge & Office | Y | none (inherited) | custom (new) | 12 | allowlist | 🟡 | created (P2) |
| 21 | local-ai-doctor | AI-Infra | Y | none | custom | 28 | default (to be allowlisted) | 🟡 | keep |
| 22 | local-ai-guardian | AI-Infra | Y | none | custom | 57 | default (to be allowlisted) | 🟢 | keep |
| 23 | local-ai-guardian.backup.20261103-142924 | AI-Infra | Y | none | custom | 2263 | default | 🟢 | keep pending (Non-Goal 3) |
| 24 | office-bot | Knowledge & Office | Y | none (inherited) | custom (new) | 12 | allowlist | 🟡 | created (P2) |
| 25 | orchestrator | AI-Infra | Y | none | custom | 25 | default (to be allowlisted) | 🟢 | keep (hub router) |
| 26 | planner | code-dev | — | none | generic (to fix) | 15 | default | 🔴 | keep |
| 27 | tech-teacher-researcher | AI-Infra | — | none | custom (new) | 12 | allowlist | 🟡 | created (P2, renamed from researcher) |
| 28 | researcher-bot | AI-Infra | Y | none | custom | 29 | default (to be allowlisted) | 🟢 | keep |
| 29 | restaurant-wine-lead | HCREALTY | — | none | generic (to fix) | 15 | default | 🔴 | keep (F&B sub-bot) |
| 30 | reviewer | code-dev | — | none | generic (to fix) | 15 | default | 🔴 | keep |
| 31 | sales-marketing-director | HCREALTY | — | none | generic (to fix) | 15 | allowlist | 🟡 | keep (Sales & Mktg lead) |
| 32 | sous-chef-pastry | HCREALTY | — | none | generic (to fix) | 15 | default | 🔴 | keep (F&B sub-bot) |
| 33 | spa-hammam-manager | HCREALTY | — | none | generic (to fix) | 15 | default | 🔴 | deferred (later batch) |
| 34 | splash-7777 | AI-Infra | Y | none | generic (to fix) | 16 | default | 🟡 | keep pending |
| 35 | wellness-director | HCREALTY | — | none | generic (to fix) | 15 | default | 🔴 | excluded (snapshot taken; deletion pending confirmation) |
| 37 | skills-manager | AI-Infra | Y | none (inherited) | custom (new) | 12 | allowlist | 🟡 | created (P2, Skills Manager CEO) |

## Notes

- `cfg` = has profile config.yaml (Y) or inherits global (—).
- Model pins: all removed in P3 (2026-11-26); global default set to `MLX-Qwen3.5-9B-DeepSeek-V4-Flash-4bit` on `omlx` (validated 9B local). Aux slots remain to be pinned to 2.6B (P3 follow-up).
- Skills counts are as of P1; new bots (librarian-bot, office-bot) seeded with 12 bundled skills.
- Readiness reflects pre-P2 state for existing bots; new bots are 🟡 (MVB written, acceptance unverified).
- Room membership (FR-8) unchanged: 4 rooms; librarian-bot/office-bot proposed for a new "Knowledge & Office" room (to be authorized).
EOF
echo "written"; wc -l /Users/user/Documents/AI_AGENTS_TOOLS/Hermes_Desktop/hermes-bot-fleet-sot/registry.md