# Orchestrator Bot — Design Spec

> **Architectural spec** for the Hermes orchestrator bot: the first-door dispatcher that routes missions to a roster of specialist bots.

**Date:** 2026-09-28
**Status:** Approved for implementation
**Implements:** User request — "design the orchestrator that delegates to dev-coder and future bots"

---

## 1. Intent

### What it is
A new Hermes bot profile — `~/.hermes/profiles/orchestrator/` — that lives in the Bots menu alongside dev-coder. It is the user's first door into the Hermes bot ecosystem. The user talks to the orchestrator; the orchestrator talks to the bots.

### What it does
1. Receives a mission from the user
2. Understands intent and scope
3. Matches the mission to one or more bots from its roster
4. Delegates each phase to the appropriate bot's profile
5. Collects results, verifies completion, reports back to the user
6. Records what worked and what didn't; suggests upgrades when patterns emerge

### What it does NOT do
- Write code, read PDFs, run searches, or perform domain work itself
- Modify bot profiles autonomously — it suggests; the human applies
- Allow bots to talk to each other directly — the orchestrator is the hub

### Success criteria
- A user can say "build a rate-limited API" and the orchestrator routes it to dev-coder without being told which bot to use
- A user can say "research X, then build Y, then write a doc" and the orchestrator chains researcher-bot → dev-coder → office-manager without being told the chain
- Adding a new bot to the roster requires only a roster card, not a code change to the orchestrator
- The orchestrator never pretends to do work it doesn't have skills for

---

## 2. Architecture

### Profile structure (mirrors dev-coder)

```
~/.hermes/profiles/orchestrator/
├── config.yaml          # Inherits LM Studio model from main profile (same pattern as dev-coder)
├── profile.yaml         # Bot Mode marker — marks this profile as a Bot
├── SOUL.md             # Personality: sharp dispatcher, routes not does, asks when ambiguous
└── skills/
    └── orchestrator/
        └── SKILL.md    # Routing logic, roster format, delegation rules, upgrade triggers
```

 config.yaml inherits the model the same way dev-coder's does — same provider, same model, no separate credential setup.

### Roster model

The orchestrator's routing brain is a **roster of bot cards**. Each card is a small structured descriptor. The orchestrator reads all cards each time it routes.

**Card schema:**

```
Bot card:
- name: "dev-coder"                    # Human-readable name (shows in Bots menu)
- profile: "~/.hermes/profiles/dev-coder"  # Path to the bot's Hermes profile
- domain: "code — build, modify, debug, review, test"  # What this bot is for
- skills_loaded: ["spec-driven-development", "test-driven-development", "code-review-and-quality"]  # Skills the bot loads for its work
- tools: ["opensrc", "patch", "write_file", "terminal", "search_files", "execute_code", "read_file"]  # Tools the bot has
- trigger_keywords: ["build", "fix", "debug", "review", "feature", "refactor", "tests", "code"]  # Keywords that suggest this bot
- autonomy: "full"                    # "full" = runs to completion and reports back; future: "stepwise" = report per step
- version: "1"                        # Roster card version — bump when the card changes meaningfully
```

**v1 roster (pre-populated):**

| Bot | Domain | Trigger keywords |
|---|---|---|
| dev-coder | Code: build, modify, debug, review, test | build, fix, debug, review, feature, refactor, tests, code |
| researcher-bot (placeholder) | Research: web search, document retrieval, synthesis | research, investigate, look up, find, compare |
| tech-teacher (placeholder) | Explain: concepts, code walkthroughs, tutorials | explain, teach, walk me through, how does, what is |
| skills-master-manager (placeholder) | Skills: create, update, manage Hermes skills | skill, create skill, update skill, manage skill |
| office-manager (placeholder) | Office docs: xlsx, pdf, docx, pptx, csv, md | spreadsheet, excel, pdf, word, document, presentation, csv |

Placeholder bots have a card but no profile yet. The orchestrator routes to them; the human creates the profile when ready. This is intentional — the roster grows as the bots are built, not all at once.

**Card storage:**

The roster lives in the orchestrator's skill directory as a single file:

```
~/.hermes/profiles/orchestrator/skills/orchestrator/roster.yaml
```

Why a file and not memory? Memory is for learned facts across sessions. The roster is configuration — it's authoritative, versioned, and editable by the human. Memory is for the orchestrator's notes *about* the roster (e.g., "dev-coder keeps missing spec-driven workflow — suggest loading spec-driven-development").

### Mission flow

```
User says: "Build a rate-limited REST API in Rust"

Orchestrator:
  1. Parse intent: this is a code-build mission
  2. Scan roster cards: dev-coder matches (domain=code, trigger_keywords contains "build")
  3. Check for multi-bot need: mission is "build X" — single bot is enough
  4. Delegate to dev-coder:
     - Method: spawned hermes process (full autonomy, runs to completion)
     - Context passed: mission description, project context, any relevant memory notes
     - dev-coder loads its own skills (spec-driven-development, test-driven-development, code-review-and-quality)
  5. Wait for dev-coder to finish (or timeout)
  6. Read result:
     - If complete and verified: report to user
     - If incomplete: re-delegate with tighter constraints, or escalate to user
     - If wrong: escalate to user with diagnosis
  7. Record in orchestrator memory: mission type, bot used, outcome, any notes

User says: "Research gRPC, then build a Rust server, then write a product doc"

Orchestrator:
  1. Parse intent: multi-phase mission — research, build, document
  2. Decompose into phases:
     - Phase 1: research gRPC → researcher-bot
     - Phase 2: build Rust server from research → dev-coder
     - Phase 3: write product doc from built server → office-manager
  3. Delegate phase 1 to researcher-bot (spawned process)
  4. Collect result → pass summary to phase 2
  5. Delegate phase 2 to dev-coder (spawned process) with researcher-bot's summary as context
  6. Collect result → pass to phase 3
  7. Delegate phase 3 to office-manager (spawned process)
  8. Synthesize final report: what was researched, what was built, what doc was written
  9. Report to user
```

### Delegation mechanism

Hermes provides two delegation paths. The orchestrator uses both:

| Path | Tool | When | What it gives the bot |
|---|---|---|---|
| Spawned process | `terminal` with `hermes chat -q` or tmux session | Full bot run, autonomous to completion, hours | Independent Hermes process, full tool access, own session, own memory |
| Subagent | `delegate_task` | Quick subtask, bounded scope, minutes, parallelizable | Isolated subagent context, subset of parent tools, shares parent process |

**Default for v1: spawned process.** The "run autonomously to completion and report back after testing/debugging" model you described maps directly to a spawned `hermes` process. The orchestrator fires the bot, waits, reads the result, and decides.

**delegate_task is for lighter weight.** When the orchestrator itself needs to split a task into parallel subtasks without standing up full bot sessions — e.g., "search three sources in parallel" — it uses delegate_task. The subagents are not full bots; they're workers the orchestrator spawns and collects from.

**Context passing:**

The orchestrator passes to the target bot:
- The mission description (what to do)
- Any project context the user provided (repo path, constraints, preferences)
- Relevant memory notes from the orchestrator's own memory (e.g., "user prefers TDD on new features")
- The output of any prior phase in a multi-bot chain

The target bot loads its own skills and tools from its own profile. The orchestrator does not load them on the bot's behalf.

### Skills: orchestrator's own vs. target bots'

**Orchestrator's skills (its own skill directory):**

- `orchestrator/SKILL.md` — the core: how to read a mission, match it to roster cards, decide single vs. multi-bot, choose delegation method, handle results, and suggest upgrades. This is the orchestrator's playbook.

That's it for v1. The orchestrator does not need spec-driven-development, test-driven-development, or code-review-and-quality — it doesn't write code. It routes.

**Target bot skills: live in the target bot's profile.** When dev-coder runs, it loads its own skills from `~/.hermes/profiles/dev-coder/skills/`. When researcher-bot runs, it loads its own. The orchestrator never loads another bot's skills.

This is the key structural decision: **the orchestrator is a router with a roster of cards, not a coordinator that knows how to do the work.** Adding a new bot is adding a card, not teaching the orchestrator a new skill.

### Memory: what each party remembers

**Orchestrator memory (its own):**
- Mission history: what was asked, which bot(s) handled it, what the outcome was
- Routing notes: when a bot was a good fit, when it wasn't
- Upgrade suggestions: patterns the orchestrator noticed ("dev-coder missed spec-driven workflow on 3 of the last 5 feature builds")
- User preferences learned across missions (e.g., "user wants tests on every feature")

**Target bot memory (its own, separate):**
- Each bot has its own memory in its own profile
- The orchestrator can read a bot's memory before delegating to decide what context to pass
- The orchestrator writes to its own memory after a mission; it does not write to the bot's memory

**Why separate:** context isolation. The orchestrator's memory is about routing and outcomes. Dev-coder's memory is about code and project context. Mixing them would bloat both.

---

## 3. Upgrade Triggers

Two sources, as specified:

### Orchestrator-driven upgrades

The orchestrator notices patterns across missions and suggests changes:

| Pattern | Suggestion |
|---|---|
| Bot returns structurally wrong result for mission type (e.g., code without tests when tests were expected) | "Bot X produced Y but the mission asked for Z. Suggest adding constraint to next delegation, or loading skill S." |
| Bot repeatedly fails on a class of task | "Bot X has failed on N missions of type T. Review roster card — consider adding skill, adjusting trigger_keywords, or creating a specialist bot." |
| Mission type appears that no bot handles well | "No bot in roster matches this mission type. Suggest creating a new bot, or adding trigger_keywords to an existing bot." |
| Bot consistently produces good results on a mission type | "Bot X handled N missions of type T well. Consider widening its trigger_keywords to catch similar missions automatically." |

These are suggestions, recorded in orchestrator memory and surfaced to the human. The orchestrator does not apply them autonomously in v1.

### Human-driven upgrades

The human makes changes directly:

| Action | Effect |
|---|---|
| Human adds a new bot profile + card | Next mission routing includes the new bot |
| Human edits a roster card (skills_loaded, trigger_keywords, domain) | Next mission routing uses the updated card |
| Human removes a bot from the roster | Orchestrator stops routing to it |
| Human confirms an orchestrator suggestion | Orchestrator applies the change (card edit, skill add, etc.) |

**Upgrade path is: card edit → next mission uses updated card.** No reload, no restart, no downtime. The card is data the orchestrator reads each time it routes.

### Memory upgrade

When a skill is added to a bot or a new need emerges, the orchestrator updates its memory:
- Records the change (what was added, why, when)
- Links the change to mission outcomes (did the added skill improve results?)
- Surfaces stale entries for review (e.g., "this upgrade suggestion is 30 days old with no missions since — still relevant?")

---

## 4. What's in v1 vs. later

### In v1

- Orchestrator profile: config.yaml, profile.yaml, SOUL.md, orchestrator/SKILL.md
- Roster: roster.yaml with dev-coder card + placeholder cards for researcher-bot, tech-teacher, skills-master-manager, office-manager
- Routing: keyword/domain match against roster cards
- Delegation: spawned `hermes` process (full bot runs) + delegate_task (lightweight parallel subtasks)
- Memory: orchestrator memory for mission history, routing notes, upgrade suggestions
- Upgrade suggestions: orchestrator notices patterns, surfaces to human
- Human applies upgrades by editing cards

### Not in v1 (future)

- Automatic bot provisioning (a bot that creates bot profiles — that's the skills-master-manager's job, built separately)
- Bot-to-bot direct communication (orchestrator stays the hub)
- Consensus/voting between bots (one primary bot per phase; review is a second invocation, not a vote)
- Autonomous profile modification (orchestrator suggests, human applies — flip this later if wanted)
- Sophisticated routing beyond keyword/domain match (e.g., semantic similarity, confidence scoring — add when keyword match proves insufficient)
- Mission timeout with automatic retry (v1 reports timeout to user; retry logic comes later)
- Parallel bot phases (v1 runs phases sequentially; parallelize when there's a demonstrated need)

---

## 5. Failure modes and mitigations

| Failure | Mitigation |
|---|---|
| Orchestrator can't match mission to any bot | Ask the user which bot they want, or suggest creating one. Don't guess. |
| Target bot fails mid-mission (crashes, times out, produces garbage) | Orchestrator detects failure (process exit, timeout, result quality check), reports to user with diagnosis, offers retry or escalation. |
| Multi-bot chain breaks mid-chain (phase 2 fails after phase 1 succeeded) | Orchestrator reports what was completed and what failed. User decides whether to retry phase 2, skip to phase 3, or stop. |
| Roster card is wrong (wrong domain, missing trigger keywords) | Bot doesn't get routed to for relevant missions. Human edits the card. Orchestrator can suggest the edit if it notices the gap. |
| Delegated bot loads wrong skills for the mission | Bot's own skill-loading logic is responsible. Orchestrator passes the mission description; bot decides which skills to load. If bot consistently loads wrong skills, that's a bot-level issue, not orchestrator. |
| Orchestrator runs out of context from too many mission summaries | Orchestrator memory has a retention policy: summarize old missions, prune very old ones. Memory entry spec defines the policy. |

---

## 6. Testing the orchestrator

v1 testing is manual — give the orchestrator missions and verify it routes correctly:

1. **Single-bot routing:** "Build a CLI tool in Rust" → should route to dev-coder
2. **Multi-bot chain:** "Research X, build Y, write a doc" → should decompose and route to multiple bots
3. **No-match:** "Do something no bot handles" → should ask the user or suggest a new bot
4. **Ambiguous mission:** "Fix this" (with no context) → should ask clarifying questions
5. **Roster update:** Add a trigger keyword to dev-coder's card → next similar mission routes correctly

These are the acceptance tests. They're manual for v1; automate later if the orchestrator proves worth it.

---

## 7. Open questions (for implementation, not for design approval)

These are resolved during implementation, not before:

1. **Exact delegation command** — `hermes chat -q` vs. tmux session vs. something else. Depends on how long missions typically take and whether the user wants to watch them run.
2. **Context size limit** — how much context the orchestrator passes to a target bot before it overwhelms the bot's context window. Resolve when there's a realmission that hits the limit.
3. **Memory retention policy** — how old a mission summary can be before it's pruned. Resolve when memory gets crowded.
4. **Card format** — YAML for v1 (human-readable, editable). Could move to a structured format later if the roster grows large.

---

*End of spec.*
