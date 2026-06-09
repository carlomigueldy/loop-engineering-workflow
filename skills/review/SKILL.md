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

1. Determine the diff range: the merge-base of the feature branch with the default branch, or the first checkpoint commit of this piece of work. State the range.
2. Identify the spec path (`docs/loop/specs/...`) if one exists for this work.
3. Dispatch **one** subagent (general-purpose, read-only mindset) with the prompt template below. Do not dispatch more than one reviewer; do not have the reviewer fix anything.
4. Triage every finding (see Triage). Apply fixes yourself in the main context.
5. Re-run verification after fixes.

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

Rules:
- Read the diff and any file needed to judge it. Do not modify anything.
- Report findings ONLY for issues introduced or made worse by this diff — not pre-existing problems (note at most one line: "pre-existing issues observed: yes/no").
- For each finding: severity (blocker | important | nit), file:line, what is wrong, why it matters, and a concrete suggested fix.
- If you find nothing at a severity level, say so explicitly. An empty review of a large diff is a red flag — look harder at edge cases before concluding.

Return: a numbered list of findings grouped by severity, then a one-paragraph overall verdict.
```

## Triage

Every finding gets exactly one disposition, stated out loud before moving on:

- **Fix** — blockers and importants default here. Fix now, in this session.
- **Defer** — valid but out of scope. Record it: on Deep, append to the plan doc's end under `## Deferred from review`; otherwise tell the user. A deferral without a written destination is a silent drop — not allowed.
- **Reject** — the reviewer is wrong or the tradeoff is intentional. State the reason in one or two sentences. Rejecting a blocker requires telling the user explicitly.

Never silently ignore a finding. Never let the reviewer's verdict replace verification — the verify suite still runs after fixes.
