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
