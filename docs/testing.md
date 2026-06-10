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
