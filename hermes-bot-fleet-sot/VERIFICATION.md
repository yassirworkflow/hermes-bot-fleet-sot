# VERIFICATION (auto-run subset) — 2026-11-26

## T-01 registry count
profiles (excl dot/active_profile):       35

## T-03 memory files non-empty
  MISSING: ape-bot
  MISSING: asset-manager
  MISSING: competition-survey-manager
  MISSING: culinary-director
  MISSING: culinary-frontline-agent
  MISSING: debugger
  MISSING: deputy-agent
  MISSING: dev-coder
  MISSING: estate-accountant
  MISSING: estate-butler-supervisor
  MISSING: estate-engineer
  MISSING: estate-housekeeping-agent
  MISSING: executive-housekeeper
  MISSING: experiences-director
  MISSING: gm-agent
  MISSING: guest-experience-manager
  MISSING: guest-services-agent
  MISSING: hammam-master
  MISSING: lead-spa-receptionist
  MISSING: local-ai-doctor
  MISSING: local-ai-guardian.backup.20261103-142924
  MISSING: local-ai-guardian
  MISSING: planner
  MISSING: researcher-bot
  MISSING: restaurant-wine-lead
  MISSING: reviewer
  MISSING: sales-marketing-director
  MISSING: sous-chef-pastry
  MISSING: spa-hammam-manager
  MISSING: splash-7777
profiles missing MEMORY.md/USER.md: 30

## T-04 allowlists present
profiles without allowlist: 0

## T-05 profile-level main-model pins
profiles with model: block:        0 (expect 0)

## T-06 aux on 2.6B (dev-coder check)
dev-coder 35B in aux: 2

## T-07 credential hygiene (shared keys in profile .env)
shared-key leaks in profile .env: 0 (expect 0)

## T-02 generic SOUL check
  GENERIC: splash-7777
profiles still on generic SOUL: 1 (splash-7777 expected as generic helper)

## Global model default
model:
  default: MLX-Qwen3.5-9B-DeepSeek-V4-Flash-4bit

## group_router aux
  group_router:
    model: LiquidAI--LFM2.5-2.6B-MLX-4bit
    provider: omlx
    timeout: 30

## Post-fix re-check (2026-11-26)
- T-01: 35 active profiles (wellness pair deleted; 34 original -2 +3 new = 35).
- T-03: 0 profiles missing MEMORY.md/USER.md (59 files created).
- T-06: aux slots pinned to LiquidAI--LFM2.5-2.6B-MLX-4bit (omlx) globally; dev-coder aux 35B string count in file = 2 (provider model-catalog entries only, not aux).
- T-07: 0 shared-key leaks in profile .env (51 lines removed).
- T-02: 1 generic SOUL remaining (splash-7777, marked generic helper — allowed exception).
- Wellness profiles deleted; snapshots at hermes-snapshots/2026-11-26-wellness-predelete/.
