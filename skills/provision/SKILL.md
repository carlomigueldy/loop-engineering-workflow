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

Running provision twice in a row with no project changes in between must produce zero diff on the second run. Re-provision after a plugin update refreshes generated content while preserving the rules block and all user content outside the markers.
