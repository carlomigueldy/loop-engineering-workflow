# Engineering Loop — Design

**Date:** 2026-06-10
**Status:** Approved
**Repo:** `loop-engineering-workflow` (template/plugin source)

## Problem

The current setup — Forge Ops agent fleet, superpowers plugin, codex plugin — suffers from four problems:

1. **Token burn:** multi-agent orchestration (orchestrator → specialist → reviewer handoffs) re-pays context costs at every relay for little quality gain.
2. **Inconsistent quality:** results vary run to run; regressions and rework slip through.
3. **Too much ceremony:** the full pipeline is overkill for small tasks.
4. **Nothing composes:** three separate systems, improvised per session, no single repeatable loop.

## Goal

One self-contained engineering loop, packaged as a Claude Code plugin, provisioned into any project with a thin per-repo footprint. High quality end to end, token-efficient by design, ceremony scaled to task size. Borrows the best of superpowers (process discipline, verification evidence) and compound engineering (learnings fed back into the system) without mandating anything unnecessary — TDD is available, never required.

## Non-goals

- No dependency on the Forge Ops agents, superpowers plugin, or codex plugin. They may coexist but the loop never requires them.
- No multi-agent orchestration layer. No standing orchestrator.
- No vendoring of loop logic into projects (logic lives in the plugin; only config and state live per-repo).

## Architecture

This repo becomes a Claude Code plugin (working name **`eng-loop`**) with a marketplace manifest so it installs via `claude plugin marketplace add` pointing at this repo. Four components:

| Component | Kind | Purpose |
|---|---|---|
| `loop` | skill | The methodology: tier classification, per-tier checklists, token-discipline rules. The document every session follows. |
| `provision` | command | One-time project setup: stack detection, verify-command discovery, managed CLAUDE.md section, `docs/loop/` scaffolding. Idempotent. |
| `compound` | skill | Learning capture: what qualifies, where each kind goes, pruning rule. Final loop step at every tier; also runnable standalone. |
| `review` | skill | Deep-tier fresh-context review: one clean subagent reads the diff cold against the spec; findings come back triaged. |

**Per-project footprint after `/provision`:**

- A managed section in the project's CLAUDE.md (delimited by markers, e.g. `<!-- eng-loop:start -->` / `<!-- eng-loop:end -->`): verify commands (test/lint/build), stack notes, pointer to the loop.
- `docs/loop/` containing `specs/`, `plans/`, `learnings/` (with `learnings/INDEX.md`).
- Nothing else. No copied agents or skills. (Project skills created later by the compound step are project-owned state, not vendored loop logic — the loop itself never lives in the repo.)

Project state (specs, plans, learnings) is committed to git, so it survives context compaction and is readable by any future session.

## The Loop

Every task starts with classification. Claude states the tier and reasoning in one line ("Tier: Standard — touches the auth flow but no schema changes") and proceeds. The user can override at any time ("make it quick" / "go deep").

### Quick — typos, config tweaks, one-liners, doc edits

1. Do it.
2. **Verify with evidence** — run the relevant verify command, show output.
3. Compound check (silent skip when nothing learned).

### Standard — the default: features, bugfixes, refactors

1. **Orient** — read only code in the change path; check `learnings/INDEX.md` for relevant topics and open only matches.
2. **Inline plan** — 3–6 bullets stated in chat. No doc file, no approval gate.
3. **Implement** — tests where they fit the change. TDD optional, never mandated.
4. **Self-review** — read the full diff before declaring done: debug leftovers, missed call sites, scope creep.
5. **Verify with evidence** — full verify command, output shown.
6. Compound check.

### Deep — architectural, risky, large, or unfamiliar territory

1. **Spec** — clarifying Q&A → committed to `docs/loop/specs/YYYY-MM-DD-<topic>.md`.
2. **Plan** — staged implementation plan → committed to `docs/loop/plans/`.
3. **Implement in stages** with a checkpoint commit per plan stage.
4. **Fresh-context review** — clean subagent reads the diff cold against the spec; findings triaged: fix / defer / reject with reason.
5. **Verify with evidence.**
6. Compound check — Deep almost always produces a learning.

### Escalation rule (the loop's error handling)

If a task surprises mid-tier — a Quick fix touches more files than expected, a Standard task reveals architectural implications — **stop, say so, re-classify upward**, and adopt the higher tier's remaining stages. Escalation needs no sign-off; de-escalation requires the user's. Misclassification stays cheap instead of catastrophic.

## Token discipline (hard rules in the `loop` skill)

- **One context drives the whole loop.** No orchestrator, no specialist handoffs. Subagents are allowed in exactly two places: Deep-tier fresh review (a clean context is the point) and parallel read-only exploration when the search space is genuinely large.
- **Read discipline:** targeted search before file reads; read only files in the change path; never re-read a file just edited.
- **Write discipline:** plans inline on Standard (doc files only on Deep where the paper trail earns its cost); no ceremonial summaries; learnings written only when real.
- **State lives in git, not context:** Deep-tier checkpoint commits let a compacted or fresh session resume from the repo, not from memory.

## Compounding

The final step at every tier asks one question: **would a future session in this repo need this?** Three homes, by kind:

| Kind | Home | Read-back |
|---|---|---|
| Rule (always-applies do/don't) | CLAUDE.md managed section | Loaded every session; hard budget ~30 lines — adding when full means pruning one |
| Gotcha / context (occasionally relevant) | `docs/loop/learnings/<topic>.md` + one line in `INDEX.md` | Standard+ Orient checks the index, opens only matching topics |
| Reusable procedure | Project skill in `.claude/skills/` | Triggered by skill description |

**Pruning is part of the contract:** a learning baked into code, CI, or types gets deleted at the next compound check that notices it is stale.

## Provisioning

`/provision` in a target project:

1. Detect stack (package.json, pyproject.toml, etc.) and discover candidate verify commands.
2. Propose them to the user for confirmation.
3. Write the managed CLAUDE.md section (create CLAUDE.md if absent; update only between markers if present).
4. Scaffold `docs/loop/` with `specs/`, `plans/`, `learnings/INDEX.md`.
5. Commit.

Re-running refreshes the managed section without touching anything else — this is how plugin updates that change the per-repo contract propagate.

## Testing

The plugin is mostly prose, so testing is empirical:

- **Loop behavior:** provision one real project (e.g. `contextly` or `doable`), run one task per tier, check against a behavior checklist: correct tier stated up front, evidence shown before any "done" claim, no doc files created on Standard, learnings land in the correct home, escalation announced when scope grows.
- **`/provision`:** real tests for idempotency (run twice → no diff the second time) and stack detection across at least two project types, plus marker-preservation (user content outside markers untouched).

## Decisions log

| Decision | Choice | Rationale |
|---|---|---|
| Relationship to existing setup | Self-contained replacement | Projects must work even if global setup drifts; "nothing composes" pain |
| Ceremony scaling | Three explicit tiers, Claude classifies, user overrides | Fixes over/under-processing; removes per-session improvisation |
| TDD | Optional, never mandated | "You don't need TDD for everything" |
| Quality gates | Verification evidence (all tiers); fresh-context review + spec/plan docs (Deep only) | Cheapest gates with highest payoff at the right tier |
| Compounding | Conditional check, every tier | Zero ceremony when nothing learned; captures gotchas found even in quick fixes |
| Packaging | Plugin + thin `/provision` (Approach A) | One user, ~14 projects: drift is the killer problem; one fix updates all projects |
| Execution model | Single-context-first; subagents only for cold review and large read-only exploration | Direct answer to the token-burn pain of agent-to-agent relays |
