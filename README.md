# eng-loop

A three-tier engineering loop for Claude Code. One plugin, installed once, used across every project — each repo gets only a thin provisioned footprint (a managed CLAUDE.md section + `docs/loop/` state).

## The loop

| Tier | For | Stages |
|---|---|---|
| **Quick** | Typos, config tweaks, one-liners | do it → verify with evidence → compound check |
| **Standard** | Most features, bugfixes, refactors | orient → inline plan → implement → self-review → verify → compound |
| **Deep** | Architectural, risky, unfamiliar | spec → plan → staged implementation → fresh-context review → verify → compound |

Claude states the tier up front; you can override anytime. Escalation upward is free; de-escalation needs your sign-off. Never "done" without command output as evidence. TDD is available, never mandated.

## Install

```bash
# from a local clone
claude plugin marketplace add ~/personal/loop-engineering-workflow
claude plugin install eng-loop@eng-loop-marketplace

# or from GitHub (once this repo is pushed to github.com/carlomigueldy/loop-engineering-workflow)
claude plugin marketplace add carlomigueldy/loop-engineering-workflow
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
