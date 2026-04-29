---
name: release
description: Use this skill when the user invokes `/release-flow:release` or asks to ship, cut, tag, or publish a release of the current project. Drives the full release workflow — version bump, tests, docs review, release-notes draft with approval gate, commit, push, and GitHub release. Trigger phrases include "ship a release", "cut version X", "release this", "release v2.1", "tag a release", "publish a version", "make a release", "bump and tag", "/release", "/release-flow:release". Does NOT trigger for partial release steps in isolation ("just bump the version", "only push to npm", "only create the tag") — those are direct tool calls, not the full flow. Auto-detects project context for typical projects; reads optional per-project config from `.claude/release.config.json` and lifecycle markdown overrides in `.claude/release/`.
argument-hint: "[version-override-or-empty]"
allowed-tools:
  - Read
  - Edit
  - Write
  - Grep
  - Glob
  - Bash
  - AskUserQuestion
  - Skill
  - TaskCreate
  - TaskUpdate
  - TaskList
---

# /release-flow:release

Drive a software release end-to-end. The flow is: detect project context → run preflight → bump version → run tests → review docs → draft release notes → get approval → commit → push → create GitHub release → run post-release hooks.

The user invoked `/release-flow:release`. Optional argument: a version override (e.g., `/release-flow:release 2.1.0`). If absent, compute the next version automatically.

## Workflow setup — Create task list for visible progress

**Before starting Phase 0, enumerate every phase as a task** so the user sees a live checklist tick off as the workflow proceeds. Use `TaskCreate` to add the following tasks in order (one call per task):

1. "Detect project context" (Phase 0)
2. "Run preflight checks" (Phase 1)
3. "Bump version" (Phase 2)
4. "Run tests" (Phase 3)
5. "Review documentation" (Phase 4)
6. "Draft release notes" (Phase 5)
7. "Approval gate" (Phase 6)
8. "Update CHANGELOG" (Phase 7)
9. "Run pre-commit hook" (Phase 8)
10. "Commit and push" (Phase 9)
11. "Create GitHub release" (Phase 10)
12. "Run post-release hook" (Phase 11)
13. "Confirm completion" (Phase 12)

For each phase below: call `TaskUpdate` with `status: "in_progress"` when you START the phase, and `TaskUpdate` with `status: "completed"` when you FINISH it. If a phase is genuinely a no-op for this project (e.g., no `preflight.md` exists and no `preflightCmd` configured), still mark it completed — don't skip the status update.

If the workflow aborts (failed tests, user denies approval, preflight blocks), leave the current task in `in_progress` and surface the reason. Do not mark it completed when it didn't complete.

## Phase 0 — Detect project context

Read project state to figure out version file, format, test command, and repo. Prefer `.claude/release.config.json` if present; otherwise auto-detect.

1. **Check for explicit config**: `Read .claude/release.config.json` if it exists. Schema is in `references/config-schema.md`. Values from the config win over auto-detection.

2. **Auto-detect** for any field not in config. Read `references/version-formats.md` for the priority list and detection logic. If two sources disagree (e.g., a `package.json` `"version"` and a `Cargo.toml` `[package].version` differ), use `AskUserQuestion` to ask which is canonical — do NOT guess.

3. **Repo URL**: read `git remote get-url origin`. Parse to `OWNER/REPO`. If the user is on a fork, this is still correct because release tags push to `origin`.

4. **Test command**: from config, or detect from project files (see `references/version-formats.md`). If no test command can be inferred and no config provides one, ask the user — do NOT skip tests silently.

5. **Print a one-line context summary** so the user sees what was detected before any changes happen:
   ```
   release-flow: project=<name>, version=<current> (<format>), test=<cmd>, repo=<owner/repo>
   ```

## Phase 1 — Splice in project preflight

Two preflight inputs may exist; run both if both are present.

1. **`preflightCmd` from `.claude/release.config.json`** (deterministic check). If the config sets `preflightCmd`, run it as a single shell command. Non-zero exit aborts the release. Use this for purely deterministic checks (lint, format-check) that don't need Claude to reason about output.

2. **`.claude/release/preflight.md`** (Claude-led check). If this file exists, **Read it and follow its instructions before continuing**. Treat its content as authoritative project-specific guidance — it may add integrity checks (e.g., install/uninstall parity), additional doc reviews, or required pre-release verifications. If preflight reports unfixable issues, stop the release and surface them.

If neither input exists, skip this phase.

## Phase 2 — Bump version

1. Compute the next version. If the user passed an explicit version argument, use that (after validating it matches the project's format). Otherwise compute per `references/version-formats.md`:
   - **semver**: inspect commits since the previous tag and propose a bump using Conventional Commits. If any commit body has `BREAKING CHANGE:` or any subject has `!:` (e.g., `feat!:`, `fix!:`) → propose **major**. Else if any subject starts with `feat:` or `feat(scope):` → propose **minor**. Else → propose **patch**. Then use `AskUserQuestion` to confirm (options: the proposal, the next-larger bump, the next-smaller bump, custom). Pre-1.0 projects (`0.x.y`) often treat minor as breaking — surface that in the question. Don't silently bump.
   - **calver-build** (`YYYY.M.BUILD`): if YYYY.M matches today, increment BUILD; else reset to `YYYY.M.1`.
   - **calver-month** (`YYYY.MM` or `YYYY.MM.DD`): use today's date.

2. Update the version file using `Edit`. Use the pattern from config (or auto-detected) to match exactly the line that holds the version, so you change only that line.

3. Verify with a `Grep` after the edit that the new version appears exactly once at the expected location.

## Phase 3 — Run tests

Run the detected/configured test command. **Stop immediately if any tests fail.** Do not proceed with a broken build. If the test output is noisy, tail the last failure and present it to the user; the user fixes, then re-invokes the skill.

If config sets `test: "skip"` or test command is `null` and the user explicitly opted in to skipping (via config), document the skip in the release notes. Default behavior is: tests must pass.

## Phase 4 — Review documentation

Check that documentation reflects the current code. The generic checks:
- `README.md` features list matches actual functionality
- Any `docs/*.md` files reference current commands and flags
- New features added since the last release have docs

If `.claude/release/preflight.md` already covered docs, skip the redundant pass — the project's preflight is more specific. Otherwise do the generic pass and fix any drift you find before continuing.

## Phase 5 — Draft release notes

1. Fetch tags from origin (releases tagged on GitHub may not be local):
   ```bash
   git fetch --tags origin 2>/dev/null
   ```
2. Find the previous release tag. Read the project's tag prefix from `release.config.json` (`tag.prefix`, default `"v"`). If the prefix is `"v"`:
   ```bash
   git tag --list 'v*' --sort=-version:refname | head -1
   ```
   If the prefix is empty, list all tags:
   ```bash
   git tag --sort=-version:refname | head -1
   ```
   For any other prefix, substitute it in the `--list` pattern.
3. Collect commits since that tag:
   ```bash
   git log "$PREV_TAG"..HEAD --oneline --no-merges
   ```
4. **Also check uncommitted changes** (`git diff --stat`). Some sessions accumulate working-tree changes across multiple work blocks, and skipping this misses real shipping content.
5. **Cross-check against `CHANGELOG.md`**: read the previous release's section and verify nothing in your draft was already shipped. Working-tree changes can include features from earlier sessions that already got released.

Group commits into:
- **Features** — new capabilities
- **Fixes** — bug fixes
- **Docs** — documentation changes
- **Other** — refactors, internal changes, deps

Format draft:
```markdown
## What's Changed

### Features
- ...

### Fixes
- ...

### Docs
- ...

### Other
- ...

**Full Changelog**: https://github.com/<owner>/<repo>/compare/v<PREV>...v<NEW>
```

If `.claude/release/notes-template.md` exists in the project, **Read it and use it as the template for the notes** — it may define required sections (e.g., "Migration Notes", "Breaking Changes"), specific headings, or content callouts. Apply its structure to the grouped commits.

## Phase 6 — Approval gate

Show the draft notes to the user via `AskUserQuestion` with:
- **Approve** — proceed to commit and release
- **Edit** — user supplies revised notes; loop back here

Do NOT proceed past this phase without explicit approval.

## Phase 7 — Update CHANGELOG

Insert the approved notes as a new `## <NEW>` section at the top of `CHANGELOG.md` (after the file header, before the previous release section). Use the version *without* the `v` prefix to match conventional changelog entries.

If `CHANGELOG.md` does not exist, ask the user whether to create one — don't silently start one.

## Phase 8 — Splice in pre-commit hook

If `.claude/release/pre-commit.md` exists, Read it and follow its instructions before staging. Common uses: run a formatter, regenerate derived files, run a final lint pass.

Note: pre-commit runs *after* the CHANGELOG entry is added (Phase 7). If the project's pre-commit formatter touches CHANGELOG.md, that's fine — re-stage afterward. If you prefer to format the CHANGELOG before adding the entry, do that work in `preflight.md` instead.

## Phase 9 — Commit and push

```bash
git status                         # surface what's about to be committed
```

Stage only the files you actually changed during the release: the version file (from Phase 2), `CHANGELOG.md` (from Phase 7), and any files updated by pre-commit hooks (from Phase 8). Do NOT use `git add -A` — it risks pulling in untracked files the user didn't author.

```bash
git add <version-file> CHANGELOG.md <files-touched-by-pre-commit>
git commit -m "Release <TAG_PREFIX><NEW>"
git push
```

Show `git status` output before staging. If anything looks unexpected (untracked files you didn't author, paths you don't recognize), pause and ask.

## Phase 10 — Create GitHub release

```bash
gh release create v<NEW> \
  --title "v<NEW>" \
  --notes "$(cat <<'EOF'
<approved release notes here>
EOF
)"
```

If the project doesn't tag releases with `v` prefix, omit it (read config or follow existing tag convention).

## Phase 11 — Splice in post-release hook

If `.claude/release/post-release.md` exists, Read it and follow its instructions. Common uses: deploy docs, post to a channel, trigger a deploy pipeline, publish to a package registry.

## Phase 12 — Confirm completion

Print a one-line summary:
```
released v<NEW>: <github release url>
```

## Important rules

- Never skip tests silently. If tests fail, stop.
- Never commit or push before the user approves the release notes.
- Never bypass the user-defined preflight if it exists — it encodes project invariants.
- Never use `git add -A` for the release commit — stage specific files (version file, CHANGELOG, pre-commit changes). Always show `git status` first.
- If the user's project has unusual conventions you didn't account for, ASK — don't guess.
- If two version sources disagree, ASK — don't pick.
- Treat `.claude/release/*.md` files as authoritative project guidance. They override generic behavior in their phase.

## See also

- `references/version-formats.md` — version detection priority list, format presets, regex patterns
- `references/config-schema.md` — full `release.config.json` schema with examples
- `references/lifecycle.md` — which lifecycle markdown gets spliced when, and how to write one
