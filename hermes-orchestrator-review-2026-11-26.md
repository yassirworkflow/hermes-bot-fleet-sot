# Review & Challenge: Hermes Orchestrator Bot Brainstorm

**Date:** 2026-11-26
**Mode:** Read-only review. No files under `~/.hermes/` or the working tree were modified, deleted, or committed.
**Subject:** Design spec (`docs/superpowers/specs/2026-09-28-orchestrator-bot-design.md`), implementation plan, and built profile `~/.hermes/profiles/orchestrator/`.
**Evidence base:** live inspection of the built profile + fleet audit (`hermes-fleet-audit-2026-11-26.md`), `hermes profile list`, `hermes doctor`, actual `~/.hermes/profiles/` roster.

## Verdict

The design is **conceptually sound and well-structured** (thin router, roster-as-data, two delegation paths, clear v1 scope). The implementation is **partially complete and unverified** — the profile exists and is registered, but identity metadata, roster accuracy, privilege scoping, and all acceptance tests are unfinished or wrong. I would **not** call it production-ready yet; it is a working skeleton.

## What's good (keep)

- **Router-not-doer** separation is the right call; keeps the orchestrator thin and extensible.
- **Roster as a single YAML file** is human-editable, versionable, and needs no code change to add a bot. Good extensibility model.
- **Two delegation paths** (spawned `hermes` process for full runs, `delegate_task` for lightweight parallel subtasks) maps cleanly to mission scope.
- **Upgrade triggers** (orchestrator-driven pattern detection + human-driven card edits) are well-specified with concrete patterns.
- **v1 vs. later** scoping is disciplined and explicit; deferrals are named, not forgotten.
- **Failure-modes table** is thorough and names real failure classes.
- **Skill separation** — orchestrator loads only its own skill; target bots load theirs. Correct.
- Profile physically exists and `hermes profile list` shows it alongside dev-coder (gateway running).
- roster.yaml parses as valid YAML; dev-coder card matches a real mission ("Build a REST API in Rust with rate limiting" → dev-coder via `build`+`API`).
- assets/avatar.png present; SOUL.md is a coherent dispatcher persona (~2.5K).

## Challenges / gaps (fix before production)

### C1 — Roster references bots that do not exist (HIGH)
The roster has 6 cards. Only **2** point to real profiles: `dev-coder` and `local-ai-doctor`. The other 4 (`researcher-bot`, `tech-teacher`, `skills-master-manager`, `office-manager`) have `profile: TBD`. Additionally, `researcher-bot` actually **does** exist as a real profile (`~/.hermes/profiles/researcher-bot`, custom SOUL, 29 skills) but the card still says `TBD` — a stale/incorrect entry that will cause a failed delegation or a false "bot unavailable" answer. `tech-teacher`, `skills-master-manager`, and `office-manager` do not exist at all.
- **Impact:** orchestrator will attempt to delegate to missing profiles and fail at runtime, or (worse) hallucinate a bot.
- **Fix:** point `researcher-bot` to its real path; add an `available: false` (or `profile: null`) flag to placeholder cards and have the SKILL.md instruct the orchestrator to tell the user the bot isn't built yet instead of attempting delegation. Add a pre-flight existence check before any delegation.

### C2 — profile.yaml has no identity metadata (HIGH)
Actual file is only `ui_meta: {hermes-bots: {}}`. There is no `name`, `description`, `role`, or `version`. The plan's Task 3 claimed a Bot Mode marker "same structure as dev-coder's profile.yaml", but the result is an empty marker. Every other bot in the fleet (even minimal ones) has `name/description/role/version`.
- **Impact:** no role identity in teammate rosters/group routing; inconsistent with the rest of the fleet; harder to audit.
- **Fix:** add `name: orchestrator`, `description`, `role: orchestrator` (or `dispatcher`), `version: "1"` to match the fleet convention.

### C3 — No tool/permission allowlist; over-privileged (MEDIUM-HIGH)
config.yaml has no `tools` or `permissions` section, so the orchestrator inherits the broad default `platform_toolsets` (browser, computer_use, image_gen, cronjob, delegation, kanban, etc.). The design explicitly says the orchestrator "does not write code, read PDFs, run searches" — yet it has all of those tools. This contradicts the thin-router intent and the fleet's R7 risk (over-broad toolsets).
- **Fix:** define a minimal allowlist (terminal for delegation, memory, skill_view, clarify, read_file, web_search at most) and disable the rest, mirroring how asset-manager scopes tools. At minimum, drop browser/computer_use/image_gen/cronjob from the orchestrator.

### C4 — Credential sprawl (MEDIUM-HIGH)
orchestrator `.env` carries 16 keys (Telegram, Buzz, OpenRouter, OpenAI, FreeLLMAPI, OpenCode Zen, OMLX, …) — duplicates of keys already in the root `.env`. This violates the fleet's policy §6 (shared keys belong in root `.env`; profile `.env` only for profile-specific secrets). The design/plan never addressed secrets.
- **Fix:** consolidate shared keys to root `.env`; leave only profile-specific secrets (if any) in the profile `.env`.

### C5 — Misleading memory stub (MEDIUM)
Root `MEMORY.md` is 38 bytes ("Active Hermes profile: researcher-bot") while real memory lives in `memories/MEMORY.md` (1.9KB) + `memories/USER.md` (0.2KB). The root stub is stale/wrong and could make the orchestrator believe it *is* researcher-bot. This is the same R8 issue flagged in the fleet audit.
- **Fix:** remove or reconcile the root stub; keep a single memory location and document which one is authoritative.

### C6 — Acceptance tests were not actually run (HIGH)
The execution log admits the tmux server was unavailable and the LM Studio server unreachable from the sandbox, so manual acceptance tests (Tasks 8–9: single-bot routing, no-match, ambiguous, multi-bot chain) were never executed. The profile being "live" (appears in the menu) is not evidence that routing works. Production-readiness is assumed, not evidenced.
- **Fix:** run the 5 acceptance tests from the spec (§6) on the real machine with LM Studio up before declaring ready; capture results. At minimum, verify Test 1 and Test 4.

### C7 — Delegation mechanism is fragile / unverified (MEDIUM)
The SKILL.md proposes tmux or `hermes chat -q`. tmux was unavailable in the build environment, and `hermes chat -q` is fire-and-forget with no structured result channel for long (hours-long) runs. There is no timeout or polling fallback documented for hung delegations.
- **Fix:** verify `delegate_task` is actually in the orchestrator's toolset; document a result-handoff convention (e.g., bot writes to a known file, orchestrator polls); add a default timeout + escalation policy even in v1 (the spec defers this, but a hung delegation is a real production risk).

### C8 — SKILL.md is oversized vs. plan (MEDIUM)
The plan targeted ~85 lines (comparable to dev-coder's SKILL.md). The actual file is ~17K. The orchestrator loads this on every mission, so bloat directly costs context on every routing decision. Additionally, the orchestrator's *skill directory* contains 25 entries (12 base categories + extras like i-have-adhd, rtk-*, quality-review) — the design said the orchestrator should have a *small* skill set (just orchestrator/SKILL.md). This contradicts the thin-router intent and inflates context.
- **Fix:** trim SKILL.md to the routing playbook only; disable irrelevant base skill categories for the orchestrator (keep `orchestrator`, maybe `research`/`web`), mirroring dev-coder which disabled 27 skills (orchestrator disabled only 10).

### C9 — Identity-layer conflict not addressed (MEDIUM)
There are three competing identity layers: global SOUL ("Master-Chief-Orchestrator", 96 lines), profile SOUL (dispatcher), and the souls library. The design never states precedence. The global SOUL could override or dilute the dispatcher persona, especially since the global SOUL instructs "Use delegate_task to spawn specialized sub-agents" — overlapping with the orchestrator's own delegation rules.
- **Fix:** state identity precedence explicitly; consider trimming the global SOUL's effect for this profile (or scoping).

### C10 — No availability flag / guard for placeholder bots (MEDIUM)
Placeholder bots have no `available` field. The orchestrator has no rule to detect a missing profile before delegating. A user asking "research X" would match `researcher-bot` (TBD) and attempt a broken delegation.
- **Fix:** add `available: false` to placeholder cards and a SKILL.md rule: if the matched bot's profile is `TBD`/missing, inform the user the bot isn't built yet and offer to create it (or route to the closest available bot).

### C11 — No rollback/versioning for roster edits (LOW-MEDIUM)
The spec claims "card edit → next mission uses updated card, no downtime" but there is no backup or version control. Notably, the working directory is **not a git repo** (`git repo: no`), so the plan's Task 10 ("commit") could not have actually run — the "Spec written and committed" claim is unverified. The hermes-agent install dir is a git repo, but profiles live in `~/.hermes`, outside it.
- **Fix:** either place profiles under version control or document a manual backup procedure (copy roster.yaml before edits). Don't claim commits that didn't happen.

### C12 — Model policy drift (MEDIUM)
The orchestrator is pinned to OpenRouter Nemotron 550B (cloud/free), while `LOCAL_MODEL_POLICY.md` says the orchestrator should be a 9B local oMLX model. The design said it "inherits LM Studio model from main profile (same pattern as dev-coder)" but dev-coder uses a local 7B coder model, whereas the orchestrator actually uses cloud Nemotron. This is the R2 drift flagged in the fleet audit.
- **Fix:** ratify or replace the local-model policy and align the orchestrator's pin accordingly (local 9B if policy holds; document the exception if cloud is intended).

### C13 — No health-check / smoke-test integration (LOW)
Nothing verifies the orchestrator beyond "gateway running". There is no `hermes config check` / doctor integration or scripted smoke test.
- **Fix:** add a documented smoke test (config check + one routing probe) to the acceptance suite.

### C14 — "Memory full — consolidating stale entries" (LOW)
The execution log ends with a memory-consolidation note, suggesting the build session hit context limits. For a long-running orchestrator, memory pressure is a real reliability risk (the spec even lists "runs out of context from too many mission summaries" as a failure mode).
- **Fix:** verify the orchestrator's memory retention policy is implemented (summarize/prune old missions) rather than just specified.

## Additional observations

- **Roster has 6 cards, not 5** as the spec/plan stated (dev-coder, researcher-bot, tech-teacher, skills-master-manager, office-manager, **local-ai-doctor**). The extra card (local-ai-doctor) is a real profile and a reasonable addition, but it's undocumented in the spec. Either update the spec or remove it.
- **`researcher-bot` is mislabeled `TBD`** despite being a real, capable profile (custom 8.3K SOUL, 29 skills incl. evidence-ledger, search-strategy; env has SEARXNG/BRAVE/WEBSURFX keys). This is the single most impactful roster fix.
- **`local-ai-doctor`** is a diagnostics bot, not an obvious mission type. Consider whether it belongs in the default roster or behind an explicit "diagnose" trigger.
- **SOUL.md quality is good** (sharp dispatcher, asks when ambiguous, doesn't pretend to do work) — keep.
- The spec's success criteria are testable and well-stated; they just haven't been verified.

## Prioritized fix order (suggested)

1. C1 — fix researcher-bot path + add `available` flag + pre-flight guard (blocks correct routing).
2. C2 — populate profile.yaml identity (consistency).
3. C6 — run the 5 acceptance tests on the real machine (evidence).
4. C3/C8 — scope tools + trim skill/context bloat (least privilege + cost).
5. C4 — consolidate credentials (security).
6. C5 — reconcile memory stub (correctness).
7. C7/C10/C12 — delegation robustness, placeholder handling, model policy (reliability/policy).
8. C9/C11/C13/C14 — identity precedence, versioning, smoke tests, memory retention (hygiene).

## Bottom line

The orchestrator is a **good skeleton with a strong design** but it is **not yet verified or hardened**. The two things that would most improve confidence are (a) fixing the roster so it points at real bots (especially researcher-bot), and (b) actually running the acceptance tests on the live machine. Until then, treat it as "built but unvalidated." No changes were made by this review.

---
End of review (read-only).