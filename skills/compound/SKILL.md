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
