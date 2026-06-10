# eng-loop Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `eng-loop` Claude Code plugin — a three-tier engineering loop (Quick/Standard/Deep) with verification-evidence gates, fresh-context review, conditional compounding, and thin per-project provisioning — distributed from this repo via its own marketplace manifest.

**Architecture:** This repo is both the plugin root and its marketplace (the superpowers dev-marketplace pattern: `marketplace.json` with `source: "./"`). Four skills carry all logic: `loop` (the methodology), `provision` (project setup, slash-invoked only), `compound` (learning capture), `review` (Deep-tier cold review). Projects receive only a managed CLAUDE.md section and `docs/loop/` scaffolding — never copied logic.

**Tech Stack:** Claude Code plugin format (`.claude-plugin/plugin.json`, `skills/<name>/SKILL.md`), JSON + Markdown only. No build step. Verification via `claude plugin validate` and `claude --plugin-dir`.

**Spec:** `docs/superpowers/specs/2026-06-10-engineering-loop-design.md`

**Spec deviations and documented decisions (intentional):**
1. The spec lists `provision` as a "command". Commands (`commands/*.md`) are the legacy plugin format; skills are preferred and give the same UX (`/eng-loop:provision`). All four components are implemented as skills; `provision` sets `disable-model-invocation: true` so only the user triggers it.
2. Spec Provisioning step 2 requires user confirmation of detected commands. In non-interactive (headless) runs there is no one to ask, so provision proceeds with detections and reports them for later correction. The plan's automated tests exercise only this headless path; the interactive confirmation flow is exercised when the user first provisions a project for real.
3. The spec's loop-behavior empirical test ("run one task per tier" against the checklist) is deliberately deferred to user acceptance: the Deep tier requires interactive one-question-at-a-time Q&A that cannot be automated honestly. Task 10 delivers the checklist and the handoff; the run itself is the user's acceptance test.
4. The Deep tier includes a spec-approval gate ("get the user's approval of the spec before planning") that the spec's Deep step 1 does not state. Kept deliberately: it mirrors the brainstorm-and-approve flow this plugin was designed with, and de-risks the costliest tier.

---

### Task 1: Plugin manifest and marketplace

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `.claude-plugin/marketplace.json`
- Create: `.gitignore`

- [ ] **Step 1: Write `.claude-plugin/plugin.json`**

```json
{
  "name": "eng-loop",
  "displayName": "Engineering Loop",
  "description": "Three-tier engineering loop (Quick/Standard/Deep): verification evidence on every tier, fresh-context review and spec/plan docs on Deep, conditional compounding of learnings. Token-efficient by design — one context drives the loop.",
  "version": "0.1.0",
  "author": {
    "name": "Carlo Miguel Dy",
    "email": "carlomigueldy@yahoo.com"
  },
  "repository": "https://github.com/carlomigueldy/loop-engineering-workflow",
  "license": "MIT",
  "keywords": ["workflow", "engineering-loop", "verification", "compounding", "token-efficiency"]
}
```

- [ ] **Step 2: Write `.claude-plugin/marketplace.json`**

```json
{
  "name": "eng-loop-marketplace",
  "description": "Distribution marketplace for the eng-loop engineering workflow plugin",
  "owner": {
    "name": "Carlo Miguel Dy",
    "email": "carlomigueldy@yahoo.com"
  },
  "plugins": [
    {
      "name": "eng-loop",
      "source": "./",
      "description": "Three-tier engineering loop with verification evidence, fresh-context review, and compounding learnings."
    }
  ]
}
```

- [ ] **Step 3: Write `.gitignore`**

```
.DS_Store
```

- [ ] **Step 4: Validate both manifests**

Run: `claude plugin validate .claude-plugin/plugin.json --strict 2>&1 && claude plugin validate . 2>&1`
Expected: both exit 0 with "Validation passed". Note: once `marketplace.json` exists, `claude plugin validate .` validates ONLY the marketplace manifest — the `plugin.json` path is what validates the plugin and (later) its skills. Neither command enumerates skill names in its output.

- [ ] **Step 5: Commit**

```bash
git add .claude-plugin/ .gitignore
git commit -m "feat: add eng-loop plugin manifest and marketplace"
```

---

### Task 2: The `loop` skill

**Files:**
- Create: `skills/loop/SKILL.md`

- [ ] **Step 1: Write `skills/loop/SKILL.md` with exactly this content**

````markdown
---
name: loop
description: In a project provisioned with eng-loop (CLAUDE.md contains the eng-loop managed section), use at the start of ANY engineering task — feature, bugfix, refactor, config or doc change — or whenever the user asks for the engineering loop. Classifies the task into a tier (Quick/Standard/Deep), states it in one line, and drives the loop end to end. Also use when the user names a tier, says "go deep", "make it quick", or asks to escalate.
---

# The Engineering Loop

One loop for all engineering work. Ceremony scales with risk via three tiers. One context — yours — drives the whole loop; you never hand implementation to another agent.

**Announce the tier first.** Before any other action, state one line:
`Tier: <Quick|Standard|Deep> — <reason in a few words>`
The user can override at any time ("make it quick" / "go deep"). Their override always wins.

## Prerequisites

Read the project's CLAUDE.md managed section (between `<!-- eng-loop:start -->` and `<!-- eng-loop:end -->`). It holds the verify commands, stack notes, and loop-state paths. If the section is missing, suggest **once per session** that the user run `/eng-loop:provision`; if they decline or the session is non-interactive, proceed quietly with ad-hoc verify-command discovery and do not raise provisioning again.

## Tier Classification

| Tier | Use when | Signals |
|---|---|---|
| **Quick** | Trivial, reversible, contained | Typo, copy change, config value, doc edit; expected to touch ≤2 files; no logic branches, no API/schema/contract changes |
| **Standard** | The default — most work | Feature, bugfix, or refactor inside established patterns; bounded scope; no architectural decisions |
| **Deep** | Risky, large, or unfamiliar | New subsystem; breaking or externally-visible schema/API contract changes; auth, payments, security-sensitive paths; cross-cutting refactors; unfamiliar domain; work spanning multiple sessions |

When genuinely torn between two tiers, pick the higher one.

## Quick

1. Do it.
2. **Verify with evidence** (see Verification Evidence below) — run the verify command relevant to the change.
3. **Compound check** — ask: would a future session need something from this task that is not already visible in code, types, tests, or CI — a rule, a gotcha, or a reusable procedure? If no, skip silently (the expected outcome for most Quick tasks). If yes — REQUIRED SUB-SKILL: eng-loop:compound.

## Standard

1. **Orient** — read only the code in the change path. Check `docs/loop/learnings/INDEX.md` (if present); open only topics whose "when to read" line matches this task. Do not browse beyond the change path.
2. **Inline plan** — state a 3–6 bullet plan in chat. No doc file. No approval gate; the user course-corrects in conversation.
3. **Implement** — write tests where they fit the change. TDD is available when it helps; it is never mandated.
4. **Self-review** — read the full diff (`git diff`) before declaring anything: debug leftovers, missed call sites, dead code, scope creep.
5. **Verify with evidence** — run the full verify suite from the managed section.
6. **Compound check** — same question as Quick: anything a future session needs that is not already visible in code, types, tests, or CI? If no, skip silently. If yes — REQUIRED SUB-SKILL: eng-loop:compound.

## Deep

1. **Orient** — as Standard step 1, including the `docs/loop/learnings/INDEX.md` check (if present): Deep work needs the read-back most.
2. **Spec** — clarifying Q&A with the user, one question at a time, then write the spec to `docs/loop/specs/YYYY-MM-DD-<topic>.md` and commit it. Create a feature branch before this first commit (trunk-based repos may commit to the default branch). Get the user's approval of the spec before planning.
3. **Plan** — staged implementation plan to `docs/loop/plans/YYYY-MM-DD-<topic>.md`, committed. Each stage ends in a working state.
4. **Implement in stages** — one checkpoint commit per plan stage, naming the stage in the commit message. State lives in git, not in context: a fresh session must be able to resume from the repo alone.
5. **Fresh-context review** — REQUIRED SUB-SKILL: eng-loop:review. Triage every finding: fix / defer / reject, each with a stated reason.
6. **Verify with evidence** — full verify suite (the post-review-fix run from eng-loop:review satisfies this when nothing changed since).
7. **Compound check** — same question; Deep work almost always produces a learning. If yes — REQUIRED SUB-SKILL: eng-loop:compound; if genuinely nothing, say so in one line.

## Verification Evidence

Never claim done, fixed, or passing without running the actual command and showing its output in the same message as the claim.

- Use the verify commands from the managed CLAUDE.md section. Standard and Deep run the full set. Quick runs the relevant subset: the cheapest command(s) that could plausibly fail because of this change.
- If no listed command exercises the change: build or compile if possible; otherwise demonstrate the change working (run the app, hit the endpoint, render the page) and show that output. For changes no command or build touches at all (doc edits, UI copy, config values), showing the changed artifact in place is sufficient evidence. Reasoning alone is never evidence.
- A failing verify command means the work is not done. Report the failure honestly and continue the loop; never reword a failure as success.

## Escalation Rule (the loop's error handling)

If the task surprises you mid-tier — a Quick fix touches more files than expected, a Standard task reveals architectural implications — **stop, say so, and re-classify**. If the new classification is a higher tier, adopt its remaining stages (e.g. Standard→Deep mid-flight: write the spec for what remains — the spec-approval gate still applies — then continue). If it stays the same tier, say so and update the inline plan.

- Escalation needs no permission. Announce it and proceed.
- De-escalation requires the user's explicit sign-off.

## Token Discipline (hard rules)

- **One context drives the whole loop.** No orchestrator agents, no specialist handoffs. Subagents are allowed in exactly two places: the Deep-tier fresh review (eng-loop:review — a clean context is the point) and parallel read-only exploration when the search space is genuinely large (many files, unknown location). Never delegate implementation.
- **Read discipline:** targeted search (grep/glob) before file reads; read only files in the change path; never re-read a file you just edited.
- **Write discipline:** plans are inline for Standard — doc files only on Deep, where the paper trail earns its cost. No ceremonial summaries. Learnings only when real.
- **State lives in git:** on Deep, checkpoint commits per stage mean a compacted or fresh session resumes from the repo, not from memory.
````

- [ ] **Step 2: Validate**

Run: `claude plugin validate .claude-plugin/plugin.json --strict 2>&1 && ls skills/*/SKILL.md`
Expected: exit 0 with "Validation passed" (this path validates SKILL.md frontmatter; it does not enumerate skills); `ls` lists `skills/loop/SKILL.md`.

- [ ] **Step 3: Commit**

```bash
git add skills/loop/
git commit -m "feat: add loop skill — tiers, gates, escalation, token discipline"
```

---

### Task 3: The `compound` skill

**Files:**
- Create: `skills/compound/SKILL.md`

- [ ] **Step 1: Write `skills/compound/SKILL.md` with exactly this content**

````markdown
---
name: compound
description: Capture learnings at the end of a loop run — decide whether anything from this task would help a future session in this repo, and write it to the right home (CLAUDE.md rule, docs/loop/learnings topic, or a project skill). The compound check runs at the end of every eng-loop tier and invokes this skill on a yes; also runnable standalone via /eng-loop:compound.
---

# Compound — Learning Capture

One question decides everything: **would a future session in this repo need this?**

If no — stop. Say nothing, write nothing. Zero ceremony is the contract: most Quick and many Standard tasks produce no learning, and that is the expected outcome.

If yes — classify the learning and write it to exactly one home:

In an unprovisioned repo (no managed section, no docs/loop/), do not scaffold any home — surface the learning to the user as a suggested addition instead. Provision owns the footprint.

## The Three Homes

| Kind | Test | Home |
|---|---|---|
| **Rule** | A do/don't that applies to essentially every task in this repo | The `<!-- eng-loop:rules:start -->` block inside the CLAUDE.md managed section |
| **Gotcha / context** | Occasionally relevant — surprising behavior, a trap, a non-obvious dependency, a decision rationale | `docs/loop/learnings/<topic>.md` plus one index line |
| **Reusable procedure** | A multi-step process you would repeat verbatim (a release dance, a fixture rebuild, a migration recipe) | A project skill: `.claude/skills/<name>/SKILL.md` |

## Writing a Rule

Rules live between `<!-- eng-loop:rules:start -->` and `<!-- eng-loop:rules:end -->` in the project's CLAUDE.md. This block is loaded into every session, so it has a **hard budget of 30 lines**:

- One rule per line, imperative, concrete: `- Always regenerate types after editing schema.prisma (pnpm db:types)`.
- If the block is full, adding a rule means deleting the weakest existing one. Say which one you removed and why.
- Never edit anything else inside the managed section.
- If the repo has no rules block (not provisioned), do not create one — surface the rule to the user as a suggested CLAUDE.md addition instead.
- If it cannot compress to one line, it is a procedure, not a rule — write the procedure skill, with at most a one-line rule pointing at it.

## Writing a Gotcha

Create or append to `docs/loop/learnings/<topic>.md` where `<topic>` is a kebab-case area name (`supabase-rls`, `stripe-webhooks`, `vite-ssr`). Format inside the file:

```markdown
## <one-line title> (YYYY-MM-DD)
<2–6 lines: what surprised you, why it happens, what to do instead.>
```

Then ensure `docs/loop/learnings/INDEX.md` has exactly one line for the topic:

```markdown
- [<topic>](./<topic>.md) — <when to read this: the trigger condition, not a summary>
```

The "when to read" clause is what the loop's Orient step matches against — write it as a trigger ("touching RLS policies or auth-adjacent queries"), not a description.

## Writing a Procedure Skill

Create `.claude/skills/<name>/SKILL.md` in the project with `name` and `description` frontmatter (description = when to use it) and the steps as the body. Project skills are project-owned state — they belong to the repo, not to this plugin.

## What NOT to Capture

- Anything already visible in code, types, tests, or CI config — the repo records it better than prose can.
- Session-specific state ("we are halfway through X") — that belongs in specs/plans or commit messages.
- One-off trivia that will never recur.
- A duplicate of an existing learning — update the existing entry instead.

## Pruning (part of the contract)

Whenever this skill runs and you notice a stale learning — a rule baked into lint/CI/types, a gotcha fixed by a refactor, a procedure now scripted — delete it. For a gotcha, delete the entry; remove the topic file and its index line only when its last entry goes. Mention the deletion in one line. The system stays small because every capture pass is also a prune pass.

Concrete prune hooks (staleness in untouched files is otherwise never noticed):

- When appending to an existing topic file, re-check that file's other entries for staleness in the same pass.
- If `INDEX.md` exceeds ~20 lines, review the oldest topics (by their entries' dates) for staleness or consolidation during this pass and say what you pruned — or why nothing could go.
````

- [ ] **Step 2: Validate**

Run: `claude plugin validate .claude-plugin/plugin.json --strict 2>&1 && ls skills/*/SKILL.md`
Expected: exit 0 with "Validation passed"; `ls` lists `skills/compound/SKILL.md` and `skills/loop/SKILL.md`.

- [ ] **Step 3: Commit**

```bash
git add skills/compound/
git commit -m "feat: add compound skill — learning capture with three homes and pruning"
```

---

### Task 4: The `review` skill

**Files:**
- Create: `skills/review/SKILL.md`

- [ ] **Step 1: Write `skills/review/SKILL.md` with exactly this content**

````markdown
---
name: review
description: Deep-tier fresh-context review — dispatch ONE clean-context subagent that reads the full diff cold against the spec and returns triaged findings. Use after Deep-tier implementation completes and before final verification. Also runnable on demand via /eng-loop:review for any substantial diff.
---

# Fresh-Context Review

The implementing context is blind to its own assumptions. This skill gets one clean pair of eyes on the diff — a single subagent with no memory of the implementation — and brings back findings you must triage explicitly.

## When

- Required: Deep tier, after implementation, before final verification.
- Optional: any diff the user wants reviewed (`/eng-loop:review`).

## Procedure

1. Determine the diff range: BASE is the parent of the first implementation checkpoint commit (when spec and plan were committed before the branch was cut, this equals the merge-base with the default branch). Verify with `git log` that spec and plan commits fall outside the range and the first implementation stage falls inside it. State the range.
2. Identify the spec path (`docs/loop/specs/...`) if one exists for this work.
3. Ensure everything under review is committed — the reviewer sees only committed history. Commit it (or, for on-demand reviews, ask the user to) before dispatching.
4. Dispatch **one** subagent (general-purpose, read-only mindset) with the prompt template below. Do not dispatch more than one reviewer; do not have the reviewer fix anything.
5. Triage every finding (see Triage). Apply fixes yourself in the main context.
6. Re-run verification after fixes, and commit the review fixes as a checkpoint.

## Reviewer Prompt Template

Fill the bracketed parts; pass everything else verbatim.

```
You are reviewing a diff cold — you did not write it and have no context beyond what is provided. Be skeptical and concrete.

Repo: [absolute path]
Diff command: git diff [BASE]..HEAD
Spec (read first if present): [path to spec, or "none — review against the stated goal: <one-line goal>"]

Review for, in priority order:
1. Correctness — logic errors, broken edge cases, race conditions, missed call sites of changed signatures.
2. Spec compliance — anything the spec requires that the diff does not deliver, or scope the spec excludes that the diff adds.
3. Security — injection, authz gaps, secrets, unsafe defaults (flag only what is concretely present in the diff).
4. Maintainability — naming that misleads, dead code, duplication introduced by the diff.
5. Test coverage — changed behavior with no test exercising it (verification only runs existing tests; it cannot notice absent ones).

Rules:
- Read the diff and any file needed to judge it. Do not modify anything.
- If the diff command fails or returns an empty diff, stop and report that — do not review anything else.
- Report findings ONLY for issues introduced or made worse by this diff — not pre-existing problems (note at most one line: "pre-existing issues observed: yes/no").
- For each finding: severity (blocker | important | nit), file:line, what is wrong, why it matters, and a concrete suggested fix.
- If you find nothing at a severity level, say so explicitly. An empty review of a large diff is a red flag — look harder at edge cases before concluding.

Return: a numbered list of findings grouped by severity, then a one-paragraph overall verdict.
```

## Triage

Every finding gets exactly one disposition, stated out loud before moving on:

- **Fix** — blockers and importants default here. Fix now, in this session.
- **Defer** — valid but not being fixed now (out of scope, or too large/risky for this session). Record it: on Deep, append to the plan doc's end under `## Deferred from review`; otherwise tell the user. A deferral without a written destination is a silent drop — not allowed.
- **Reject** — the reviewer is wrong or the tradeoff is intentional. State the reason in one or two sentences.

Nits default to fix-or-reject, still stated. Deferring or rejecting a **blocker** is a gate: state the reason and wait for the user's acknowledgment before proceeding. In a non-interactive session, treat an unresolved blocker as Fix — never self-certify its rejection.

Never silently ignore a finding. Never let the reviewer's verdict replace verification — the verify suite still runs after fixes.
````

- [ ] **Step 2: Validate**

Run: `claude plugin validate .claude-plugin/plugin.json --strict 2>&1 && ls skills/*/SKILL.md`
Expected: exit 0 with "Validation passed"; `ls` lists `skills/compound/SKILL.md`, `skills/loop/SKILL.md`, `skills/review/SKILL.md`.

- [ ] **Step 3: Commit**

```bash
git add skills/review/
git commit -m "feat: add review skill — single cold-context reviewer with explicit triage"
```

---

### Task 5: The `provision` skill

**Files:**
- Create: `skills/provision/SKILL.md`

- [ ] **Step 1: Write `skills/provision/SKILL.md` with exactly this content**

````markdown
---
name: provision
description: Set up the eng-loop in the current project — detect the stack and verify commands, write the managed CLAUDE.md section, scaffold docs/loop/, and commit. Re-run after plugin updates to refresh the managed section. Idempotent.
disable-model-invocation: true
---

# Provision a Project

Sets up the thin per-repo footprint. Logic stays in the plugin; only config and state land here.

## Step 0 — Preconditions

- The project is a git repository on a branch (not detached HEAD); abort and report otherwise.
- The index is empty (`git diff --cached --quiet`) and `CLAUDE.md` / `docs/loop/` have no uncommitted changes (`git status --porcelain CLAUDE.md docs/loop/` prints nothing). If not, stop and ask the user to commit or stash first — provision must never sweep user work into its commit.

## Step 1 — Detect the stack

Inspect the project root (root only — no recursive sweeps):

| Marker | Stack | Candidate verify commands |
|---|---|---|
| `package.json` | Node | Read its `scripts` block. Map: test → `test`; lint → `lint`; build → `build`; typecheck → `typecheck` OR `type-check` (both naming conventions exist). If an aggregate script exists (`verify`, `check`, `ci`), emit the optional All: verify line with it and still fill the four slots individually. Use the lockfile to pick the runner: `pnpm-lock.yaml` → `pnpm run <s>`, `yarn.lock` → `yarn <s>`, `bun.lockb` → `bun run <s>`, else `npm run <s>` |
| `pubspec.yaml` | Flutter/Dart | Test → `flutter test`; Lint → `flutter analyze`, appending `dart format --set-exit-if-changed .` only if `analysis_options.yaml` exists; Build and Typecheck → none configured |
| `pyproject.toml` / `requirements.txt` | Python | Test → `pytest` if pytest is a dependency; Lint → `ruff check .` if ruff is; Build and Typecheck → none configured |
| `Cargo.toml` | Rust | Test → `cargo test`; Lint → `cargo clippy -- -D warnings`; Build → `cargo build`; Typecheck → none configured (the compiler covers it) |
| `go.mod` | Go | Test → `go test ./...`; Lint → `go vet ./...`; Build → `go build ./...`; Typecheck → none configured |
| `src-tauri/` alongside `package.json` | Tauri | Node commands above, plus stack note: "Rust backend in src-tauri/ — run `cargo check` there when touching it" |
| `turbo.json` / `pnpm-workspace.yaml` / `nx.json` | Monorepo | Stack note: "monorepo — root scripts fan out; scope to a package when iterating" |
| `docker-compose.yml` | — | Stack note only: "services run via docker compose" |

If multiple primary stack markers coexist at the root, fill each slot from the first table row (top to bottom) that provides it, and add one stack note naming all detected stacks.

A command that does not exist is **not invented**: record `none configured` for that slot. The loop's verification fallback handles it.

## Step 2 — Confirm

Present the detected verify commands and stack notes. If the session is interactive, ask the user to confirm or correct. If non-interactive (headless/print mode), proceed with the detections and list them in the final report for later correction.

## Step 3 — Write the managed CLAUDE.md section

**Precondition — marker integrity.** Count lines consisting solely of each of the four marker strings in CLAUDE.md (markers quoted in prose or code fences do not count). Proceed only when either (a) all four counts are zero — the append/create path — or (b) `<!-- eng-loop:start -->` and `<!-- eng-loop:end -->` each occur exactly once, in that order, with the rules pair exactly once each, in that order, between them — the replace path. In ANY other state (orphaned marker, duplicates, rules markers outside the outer pair), abort without writing or committing and report the malformed state to the user.

If CLAUDE.md does not exist at the project root, create it containing only the managed section. If it exists:

- **No markers present:** append the managed section at the end of the file, preceded by exactly one blank line.
- **Markers present (re-provision):** replace everything between `<!-- eng-loop:start -->` and `<!-- eng-loop:end -->` — EXCEPT the rules block: preserve the exact lines between `<!-- eng-loop:rules:start -->` and `<!-- eng-loop:rules:end -->` verbatim, including when empty. When regenerating the verify lines, compare each slot with the existing line: if they differ, keep the existing line and report the difference — the user may have hand-corrected detection; never silently revert their value. Never touch anything outside the outer markers.

Template (fill bracketed values; keep markers and headings exact):

```markdown
<!-- eng-loop:start -->
## Engineering Loop

This project uses the eng-loop plugin. Every engineering task starts by classifying a tier — see the `eng-loop:loop` skill. Managed section: hand-edit only the verify lines and the rules block; re-run /eng-loop:provision to refresh the rest.

### Verify commands
- Test: [command or "none configured"]
- Lint: [command or "none configured"]
- Build: [command or "none configured"]
- Typecheck: [command or "none configured"]
- All: [aggregate command — include this line only when an aggregate script exists]

### Stack notes
[1–4 bullet lines from detection, using the exact quoted strings from the table where given; if none, a single line: - none]

### Rules
<!-- eng-loop:rules:start -->
<!-- eng-loop:rules:end -->

### Loop state
- Specs: `docs/loop/specs/` · Plans: `docs/loop/plans/` · Learnings: `docs/loop/learnings/` (check `INDEX.md` during Orient)
<!-- eng-loop:end -->
```

## Step 4 — Scaffold docs/loop/

Create (skip any that already exist — never overwrite):

- `docs/loop/specs/` and `docs/loop/plans/` (add an empty `.gitkeep` in each)
- `docs/loop/learnings/INDEX.md` with exactly:

```markdown
# Learnings Index

One line per topic — `- [topic](./topic.md) — <when to read it>`. Added by the eng-loop compound step.
```

## Step 5 — Commit and report

If nothing changed (a no-op re-run), skip the commit and say so — the idempotency contract requires zero commits on a no-op run. Otherwise commit what provision touched:

```bash
git add CLAUDE.md docs/loop/
git commit -m "chore: provision eng-loop (managed CLAUDE.md section + docs/loop scaffolding)"
```

Then report: detected commands, stack notes, files created vs preserved, and (if non-interactive) a reminder to correct any wrong detection by editing the verify lines or re-running interactively.

## Idempotency contract

Running provision twice in a row with no project changes in between must produce zero diff on the second run. Re-provision after a plugin update refreshes generated content while preserving the rules block, any hand-corrected verify lines, and all user content outside the markers.
````

- [ ] **Step 2: Validate**

Run: `claude plugin validate .claude-plugin/plugin.json --strict 2>&1 && ls skills/*/SKILL.md`
Expected: exit 0 with "Validation passed"; `ls` lists all four: `skills/compound/SKILL.md`, `skills/loop/SKILL.md`, `skills/provision/SKILL.md`, `skills/review/SKILL.md`.

- [ ] **Step 3: Commit**

```bash
git add skills/provision/
git commit -m "feat: add provision skill — stack detection, managed section, scaffolding"
```

---

### Task 6: README

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write `README.md` with exactly this content**

````markdown
# eng-loop

A three-tier engineering loop for Claude Code. One plugin, installed once, used across every project — each repo gets only a thin provisioned footprint (a managed CLAUDE.md section + `docs/loop/` state).

## The loop

| Tier | For | Stages |
|---|---|---|
| **Quick** | Typos, config tweaks, one-liners | do it → verify with evidence → compound check |
| **Standard** | Most features, bugfixes, refactors | orient → inline plan → implement → self-review → verify → compound |
| **Deep** | Architectural, risky, unfamiliar | orient → spec → plan → staged implementation → fresh-context review → verify → compound |

Claude states the tier up front; you can override anytime. Escalation upward is free; de-escalation needs your sign-off. Never "done" without command output as evidence. TDD is available, never mandated.

## How it works

![How eng-loop works](assets/how-it-works.svg)

Install once, provision each project, then every task runs the same loop: classify the tier, do the tier's stages, pass the evidence gate, and let the compound check feed anything worth keeping back into the project state — which the next task reads at orient. That feedback edge is the point: every task can make the next one smarter.

*The diagram is editable — open [`assets/how-it-works.excalidraw`](assets/how-it-works.excalidraw) at [excalidraw.com](https://excalidraw.com).*

## Install

```bash
claude plugin marketplace add carlomigueldy/loop-engineering-workflow
claude plugin install eng-loop@eng-loop-marketplace
```

Or from a local clone:

```bash
git clone https://github.com/carlomigueldy/loop-engineering-workflow.git
claude plugin marketplace add ./loop-engineering-workflow
claude plugin install eng-loop@eng-loop-marketplace
```

## Provision a project

Inside any project:

```
/eng-loop:provision
```

Detects the stack and verify commands, writes the managed CLAUDE.md section, scaffolds `docs/loop/{specs,plans,learnings}`, commits. Idempotent — re-run after plugin updates.

## Skills

- `eng-loop:loop` — the methodology; auto-triggers at the start of engineering tasks in provisioned projects, and suggests `/eng-loop:provision` once per session where the managed section is missing
- `eng-loop:provision` — project setup (slash-only)
- `eng-loop:compound` — learning capture: rules → CLAUDE.md (30-line budget), gotchas → `docs/loop/learnings/`, procedures → project skills; pruning built in
- `eng-loop:review` — Deep-tier cold-context diff review with explicit fix/defer/reject triage

## Notes

- Self-contained: no dependency on other plugins or global agents.
- Bare `/loop` is Claude Code's built-in recurring-task skill — this plugin's skills are always namespaced `/eng-loop:...`.
- If you also run process plugins with overlapping triggers (e.g. superpowers), consider disabling them in provisioned projects to avoid double ceremony.

## Development

```bash
claude plugin validate .claude-plugin/plugin.json --strict   # validates plugin manifest + skills
claude plugin validate .                                     # validates the marketplace manifest
claude --plugin-dir .                                        # load this working copy into a session
```

## License

MIT
````

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add README with install, provisioning, and loop overview"
```

---

### Task 7: Structural validation of the full plugin

**Files:** none created — verification only.

- [ ] **Step 1: Strict validation of both manifests**

Run: `claude plugin validate .claude-plugin/plugin.json --strict 2>&1 && claude plugin validate . 2>&1`
Expected: both exit 0 with "Validation passed"; no errors or warnings. (Validate output does not enumerate skills — component inventory is checked in Step 2.)

- [ ] **Step 2: Marketplace add + install round-trip, then clean up**

```bash
claude plugin marketplace add /Users/carlomigueldy/personal/loop-engineering-workflow
claude plugin install eng-loop@eng-loop-marketplace
claude plugin list | grep -i eng-loop
claude plugin details eng-loop
```
Expected: marketplace adds without error; install succeeds; list shows `eng-loop` enabled; `details` shows all four skills (`loop`, `compound`, `review`, `provision`) in the component inventory — this is where provision's presence is verified, since it is deliberately invisible to the model (Step 3).

Then uninstall so Tasks 8–9 test the working copy via `--plugin-dir` without a duplicate registration (the installed copy is a cache snapshot and would not pick up fixes):

```bash
claude plugin uninstall eng-loop -y
claude plugin marketplace remove eng-loop-marketplace
```
Expected: both succeed; `claude plugin list | grep -i eng-loop` now returns nothing. (Final install for daily use happens in Task 10.)

- [ ] **Step 3: Headless skill-visibility smoke check**

Run:
```bash
claude --plugin-dir /Users/carlomigueldy/personal/loop-engineering-workflow --strict-mcp-config -p "Do you have skills named eng-loop:loop, eng-loop:compound, and eng-loop:review available? Answer only YES or NO followed by any missing names."
```
Expected: `YES`. Do NOT include `eng-loop:provision` in this check — it sets `disable-model-invocation: true`, so it is invisible to the model by design and would always report missing; its presence was verified via `claude plugin details` in Step 2. If this check reports a visible skill missing, that is a structural bug to fix — never "fix" it by removing `disable-model-invocation` from provision.

- [ ] **Step 4: Record results**

If any check fails, fix the structural issue, re-run, and amend the relevant commit. Do not proceed to Task 8 with failing validation.

---

### Task 8: Empirical test — provision a pnpm/turbo monorepo (contextly)

**Files (in `/Users/carlomigueldy/personal/contextly`):**
- Modify: `CLAUDE.md` (managed section appended — contextly already has a CLAUDE.md)
- Create: `docs/loop/specs/.gitkeep`, `docs/loop/plans/.gitkeep`, `docs/loop/learnings/INDEX.md`

- [ ] **Step 1: Create a test branch in contextly**

```bash
cd /Users/carlomigueldy/personal/contextly
git switch -c eng-loop-provision || { echo "branch exists — review or delete it (git branch -D eng-loop-provision) before re-running"; false; }
```

- [ ] **Step 2: Run provision headlessly with the plugin loaded**

```bash
cd /Users/carlomigueldy/personal/contextly
claude --plugin-dir /Users/carlomigueldy/personal/loop-engineering-workflow --strict-mcp-config -p "/eng-loop:provision"
```
Expected report: runner detected as `pnpm`; Test `pnpm run test`, Lint `pnpm run lint`, Build `pnpm run build`, Typecheck `pnpm run typecheck`; monorepo stack note present (turbo.json + pnpm-workspace.yaml exist); a commit created.

Note: headless sessions still inherit globally enabled plugins (superpowers process skills, the codex stop-review hook). `--strict-mcp-config` drops MCP startup cost, but inspect the first run's output for foreign-plugin interference before trusting pass/fail; if a global hook gates or distorts the run, fall back to executing `/eng-loop:provision` in a normal interactive session on this branch.

- [ ] **Step 3: Assert the managed section is well-formed**

```bash
cd /Users/carlomigueldy/personal/contextly
for m in 'eng-loop:start' 'eng-loop:end' 'eng-loop:rules:start' 'eng-loop:rules:end'; do
  test "$(grep -cF -- "<!-- $m -->" CLAUDE.md)" = 1 || echo "FAIL $m"
done
test -f docs/loop/learnings/INDEX.md && echo INDEX_OK
```
Expected: no `FAIL` lines (each marker exactly once) and `INDEX_OK`. Also confirm by eye that contextly's pre-existing CLAUDE.md content above the markers is byte-identical (visible in `git diff HEAD~1 -- CLAUDE.md` as pure addition).

- [ ] **Step 4: Idempotency — run provision again, expect zero diff AND zero new commits**

```bash
cd /Users/carlomigueldy/personal/contextly
BEFORE=$(git rev-parse HEAD)
claude --plugin-dir /Users/carlomigueldy/personal/loop-engineering-workflow --strict-mcp-config -p "/eng-loop:provision"
git status --porcelain
test "$(git rev-parse HEAD)" = "$BEFORE" && echo HEAD_UNCHANGED
```
Expected: `git status --porcelain` output is empty AND `HEAD_UNCHANGED` prints. (The HEAD check matters: provision auto-commits, so a non-idempotent second run would leave porcelain clean while silently adding a commit.)

- [ ] **Step 5: Rules-preservation check**

```bash
cd /Users/carlomigueldy/personal/contextly
perl -0pi -e 's/<!-- eng-loop:rules:start -->/<!-- eng-loop:rules:start -->\n- TEST RULE: do not delete me/' CLAUDE.md
git add CLAUDE.md && git commit -m "test: add sentinel rule"
claude --plugin-dir /Users/carlomigueldy/personal/loop-engineering-workflow --strict-mcp-config -p "/eng-loop:provision"
grep -c 'TEST RULE: do not delete me' CLAUDE.md
```
Expected: `1` — the sentinel rule survives re-provisioning.

- [ ] **Step 6: Clean up, return to main, report**

Do NOT merge. Remove the sentinel rule (revert the sentinel commit or edit it out and commit), then return the repo to its normal state:

```bash
cd /Users/carlomigueldy/personal/contextly
git checkout main
```

Report the unmerged branch name `eng-loop-provision` in contextly for the user to inspect and merge. The repo must be left checked out on `main`.

---

### Task 9: Empirical test — provision a Flutter project (doable)

**Files (in `/Users/carlomigueldy/personal/doable`):**
- Create: `CLAUDE.md` (doable has none — provision must create it)
- Create: `docs/loop/specs/.gitkeep`, `docs/loop/plans/.gitkeep`, `docs/loop/learnings/INDEX.md`

- [ ] **Step 1: Create a test branch in doable**

```bash
cd /Users/carlomigueldy/personal/doable
git switch -c eng-loop-provision || { echo "branch exists — review or delete it (git branch -D eng-loop-provision) before re-running"; false; }
```

- [ ] **Step 2: Run provision headlessly**

```bash
cd /Users/carlomigueldy/personal/doable
claude --plugin-dir /Users/carlomigueldy/personal/loop-engineering-workflow --strict-mcp-config -p "/eng-loop:provision"
```
Expected report: Flutter detected via `pubspec.yaml`; Test `flutter test`; Lint `flutter analyze` plus `dart format --set-exit-if-changed .` (doable has `analysis_options.yaml`); Build `none configured` is acceptable; CLAUDE.md created containing only the managed section; commit created. (Same foreign-plugin interference caveat as Task 8 Step 2 — interactive fallback if a global hook distorts the run.)

- [ ] **Step 3: Assert structure and idempotency**

```bash
cd /Users/carlomigueldy/personal/doable
for m in 'eng-loop:start' 'eng-loop:end' 'eng-loop:rules:start' 'eng-loop:rules:end'; do
  test "$(grep -cF -- "<!-- $m -->" CLAUDE.md)" = 1 || echo "FAIL $m"
done
BEFORE=$(git rev-parse HEAD)
claude --plugin-dir /Users/carlomigueldy/personal/loop-engineering-workflow --strict-mcp-config -p "/eng-loop:provision"
git status --porcelain                              # expect empty
test "$(git rev-parse HEAD)" = "$BEFORE" && echo HEAD_UNCHANGED
```
Expected: no `FAIL` lines; empty porcelain; `HEAD_UNCHANGED`.

- [ ] **Step 4: Return to main and report**

```bash
cd /Users/carlomigueldy/personal/doable
git checkout main
```

Leave branch `eng-loop-provision` in doable unmerged for user review, with the repo checked out on `main`. Report detection results, noting this exercised the create-CLAUDE.md path (vs contextly's append path).

---

### Task 10: Behavior checklist for live loop testing

**Files:**
- Create: `docs/testing.md` (in this repo)

- [ ] **Step 1: Write `docs/testing.md` with exactly this content**

````markdown
# eng-loop Behavior Checklist

Empirical acceptance test for the loop itself. Run in any provisioned project with the plugin installed (or `claude --plugin-dir <this repo>`). Perform one task per tier and check every box.

## Quick tier (e.g. fix a typo in a README)
- [ ] Tier stated in one line before any other action, with a reason
- [ ] No plan document created; no spec created
- [ ] Verification evidence shown (command + output) before "done"
- [ ] Compound check silent (nothing learned → nothing said beyond the work itself)

## Standard tier (e.g. small feature or bugfix)
- [ ] Tier stated up front
- [ ] Orient stayed in the change path; learnings INDEX checked (visible as a read, only matching topics opened)
- [ ] Inline plan of 3–6 bullets in chat; NO doc file created
- [ ] Full diff self-review happened before the done claim
- [ ] Full verify suite run with output shown
- [ ] If something was learned, it landed in the correct home (rule → rules block, gotcha → learnings + INDEX line, procedure → project skill) — and nothing was captured if nothing was learned

## Deep tier (pick a genuinely risky/architectural task, or simulate one)
- [ ] Tier stated up front
- [ ] Orient ran before the spec — learnings INDEX checked (if present)
- [ ] Spec written via one-question-at-a-time Q&A, committed to docs/loop/specs/, user approval obtained before planning
- [ ] Plan committed to docs/loop/plans/ with stages; checkpoint commit per stage
- [ ] Exactly ONE review subagent dispatched; findings returned with severities
- [ ] Every finding triaged out loud as fix / defer / reject with reasons; defers written to the plan doc
- [ ] Verify suite re-run after review fixes
- [ ] Compound step produced (or consciously declined) a learning

## Cross-cutting
- [ ] Escalation: start a task as Quick that secretly needs Standard (e.g. "fix this typo" where the string is referenced in tests) — loop announces re-classification upward without asking permission
- [ ] Token discipline: no orchestrator/implementation subagents appeared anywhere; no file re-read immediately after its own edit
- [ ] A failing verify command was reported as a failure, never spun as success

Record results as a dated section appended below.

## Runs

(none yet)
````

- [ ] **Step 2: Commit**

```bash
git add docs/testing.md
git commit -m "docs: add behavior checklist for live loop acceptance testing"
```

- [ ] **Step 3: Install for daily use**

Task 7 uninstalled the round-trip copy; now that all tests pass, install for real:

```bash
claude plugin marketplace add /Users/carlomigueldy/personal/loop-engineering-workflow
claude plugin install eng-loop@eng-loop-marketplace
claude plugin list | grep -i eng-loop
```
Expected: `eng-loop` installed and enabled.

- [ ] **Step 4: Hand off to the user**

The live checklist run is interactive by nature (Deep tier needs real Q&A) — per the plan's documented decision 3, it is the user's acceptance test. Tell the user: the plugin is installed, both test branches (`eng-loop-provision` in contextly and doable) await review, and the acceptance step is running one real task per tier in a provisioned project while ticking `docs/testing.md` boxes.

---

## Execution notes

- **Order:** Tasks 1→7 are strictly sequential in this repo. Tasks 8 and 9 are independent of each other (parallelizable) but require Task 7 to pass. Task 10 is last.
- **Worktree:** not needed — this repo is the plugin source, on `main`, with nothing to isolate from.
- **Rollback:** every task is one commit; `git revert` any of them. Test branches in contextly/doable are left unmerged by design, with both repos returned to `main`.
- **Headless-run hygiene:** all `claude -p` test invocations use `--strict-mcp-config` to avoid MCP startup cost, but globally enabled plugins (superpowers, codex hooks) still load. If a headless run behaves strangely, suspect foreign-plugin interference first and fall back to an interactive session before changing any eng-loop file.
- **Out of scope (YAGNI, per spec):** hooks, MCP servers, agents/ directory, multi-host manifests (.codex-plugin etc.), auto-update notifications, version tagging. Add only when a real need appears.
