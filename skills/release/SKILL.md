---
name: release
description: Use this skill when the user invokes `/claude-release:release` or asks to ship, cut, tag, or publish a release of the current project. Drives the full release workflow — version bump, tests, docs review, release-notes draft with approval gate, commit, push, and GitHub release. Trigger phrases include "ship a release", "cut version X", "release this", "release v2.1", "tag a release", "publish a version", "make a release", "bump and tag", "/release", "/claude-release:release". Does NOT trigger for partial release steps in isolation ("just bump the version", "only push to npm", "only create the tag") — those are direct tool calls, not the full flow. Auto-detects project context for typical projects; reads optional per-project config from `.claude/release.config.json` and lifecycle markdown overrides in `.claude/release/`.
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

# /claude-release:release

Drive a software release end-to-end. The flow is: detect project context → run preflight → bump version → simplify changes → run tests → review docs → draft release notes → get approval → commit → tag → push → create GitHub release → run post-release hooks.

The user invoked `/claude-release:release`. Optional argument: a version override (e.g., `/claude-release:release 2.1.0`). If absent, compute the next version automatically.

## Workflow setup — Create task list for visible progress

**Before starting Phase 0, enumerate every phase as a task** so the user sees a live checklist tick off as the workflow proceeds. Use `TaskCreate` to add the following tasks in order (one call per task):

1. "Detect project context" (Phase 0)
2. "Run preflight and security scan" (Phase 1)
3. "Bump version" (Phase 2)
4. "Simplify and review changes" (Phase 3)
5. "Run tests" (Phase 4)
6. "Review documentation" (Phase 5)
7. "Draft release notes" (Phase 6)
8. "Approval gate" (Phase 7)
9. "Update CHANGELOG" (Phase 8)
10. "Run pre-commit hook" (Phase 9)
11. "Commit, tag, and push" (Phase 10)
12. "Create GitHub release" (Phase 11)
13. "Run post-release hook" (Phase 12)
14. "Confirm completion" (Phase 13)

For each phase below: call `TaskUpdate` with `status: "in_progress"` when you START the phase, and `TaskUpdate` with `status: "completed"` when you FINISH it. If a phase is genuinely a no-op for this project (e.g., no `preflight.md` exists and no `preflightCmd` configured), still mark it completed — don't skip the status update.

If the workflow aborts (failed tests, user denies approval, preflight blocks), leave the current task in `in_progress` and surface the reason. Do not mark it completed when it didn't complete.

## Phase 0 — Detect project context

Read project state to figure out version file, format, test command, and repo. Prefer `.claude/release.config.json` if present; otherwise auto-detect.

1. **Check for explicit config**: `Read .claude/release.config.json` if it exists. Schema is in `references/config-schema.md`. Values from the config win over auto-detection.

2. **Auto-detect** for any field not in config. Read `references/version-formats.md` for the priority list and detection logic. If two sources disagree (e.g., a `package.json` `"version"` and a `Cargo.toml` `[package].version` differ), use `AskUserQuestion` to ask which is canonical — do NOT guess.

3. **Repo owner/name**: if `.claude/release.config.json` sets `repo.owner` and/or `repo.name`, use those — they override the git remote (useful for a monorepo whose release page lives in a different repo). Otherwise read `git remote get-url origin` and parse to `OWNER/REPO`. If the user is on a fork, the remote parse is still correct because release tags push to `origin`.

4. **Test command**: from config, or detect from project files (see `references/version-formats.md`). If no test command can be inferred and no config provides one, ask the user — do NOT skip tests silently.

5. **Print a one-line context summary** so the user sees what was detected before any changes happen:
   ```
   claude-release: project=<name>, version=<current> (<format>), test=<cmd>, repo=<owner/repo>
   ```

## Phase 1 — Run preflight and security scan

Up to three preflight inputs may exist; run whichever are present, in the order below — cheapest first, so a fast check aborts the release before an expensive one runs.

1. **`preflightCmd` from `.claude/release.config.json`** (deterministic check). If the config sets `preflightCmd`, run it as a single shell command. Non-zero exit aborts the release. Use this for purely deterministic checks (lint, format-check) that don't need Claude to reason about output.

2. **`.claude/release/preflight.md`** (Claude-led check). If this file exists, **Read it and follow its instructions before continuing**. Treat its content as authoritative project-specific guidance — it may add integrity checks (e.g., install/uninstall parity), additional doc reviews, or required pre-release verifications. If preflight reports unfixable issues, stop the release and surface them.

3. **`security: true` in `.claude/release.config.json`** (deep vulnerability scan). Invoke `/claude-security` via the Skill tool and ask it to scan the changes this release ships. This is the only step in the flow that depends on a *marketplace* plugin rather than a stock Claude Code install, so it needs the handling below. When `security` is absent or `false`, skip it.

   **If `/claude-security` isn't installed**, don't silently skip a gate the project opted into. Use `AskUserQuestion`:
   - **Install and re-run** — `/plugin install claude-security@claude-plugins-official`, then `/reload-plugins`, then re-invoke the release.
   - **Continue without the scan** — proceed, and document the skipped scan in the release notes the same way a skipped test run is documented.

   **Scope the scan.** `/claude-security` change scans read *committed* code only; only its full scan reads the working tree, and that costs far more.
   - Releasing from a feature branch that has commits its base lacks → ask it to scan the branch diff.
   - Releasing from the default branch, where there is no branch diff → ask it to scan the commits since the previous release tag. Phase 6 hasn't run yet, so find that tag here with the same lookup it uses (`git tag --list '<prefix>*' --sort=-version:refname | head -1`, with `tag.prefix` from config). If the lookup comes back empty — a first-ever release — there is no change scope at all.
   - Whenever a change scan can't be scoped (no prior tag, or the plugin can't take the range), do not launch an expensive run unasked. State what the alternatives cover and cost — a focused-area scan, or a full-repository scan — and let the user pick with `AskUserQuestion`, including the option to skip.

   **State the coverage gap before scanning.** Run `git status --porcelain`; if the tree is dirty, a change scan won't see those edits. Phase 3's `/simplify` also auto-applies edits *after* this phase, which the scan won't see either — the same blind spot a preflight-time `/code-review` has. Say so plainly rather than letting the release read as scanned end to end.

   **Act on the report.** The scan writes `CLAUDE-SECURITY-<timestamp>/CLAUDE-SECURITY-RESULTS.md` — read it. Findings are published only after independent verifier agents confirm them, so treat each listed finding as real: resolve it before the release continues, using the plugin's **Suggest patches** flow where that helps. If a confirmed finding can't be resolved, stop the release and surface it — do not ship a known vulnerability. The scan directory carries its own `.gitignore`, so it stays out of the release commit as long as Phase 10's named-file staging is respected.

If none of the three inputs exist, skip this phase.

## Phase 2 — Bump version

**First, reconcile any prior aborted run.** Before computing a new version, check whether the version file already holds an *uncommitted* bump from a previous release attempt that aborted (e.g., tests failed in Phase 4 and the user re-invoked). Run `git diff -- <version-file>` (and `git diff --cached -- <version-file>`); if the version value already differs from `HEAD`, do NOT bump again on top of it. Use `AskUserQuestion`:
- **Resume** — keep the existing uncommitted bump as this release's version; skip re-bumping and continue to Phase 3.
- **Reset** — `git checkout -- <version-file>` to restore HEAD's version, then bump fresh below.

Never silently bump a second time. The confirmation gate in step 1 only fires for semver; the calver formats compute the next value without a prompt, so an unreconciled re-run would skip a version (e.g., `2026.4.43` → aborted → re-run reads `.43` as current → `2026.4.44`, orphaning `.43`).

1. Compute the next version. If the user passed an explicit version argument, use that (after validating it matches the project's format). Otherwise compute per `references/version-formats.md`:
   - **semver**: inspect commits since the previous tag and propose a bump using Conventional Commits. If any commit body has `BREAKING CHANGE:` or any subject has `!:` (e.g., `feat!:`, `fix!:`) → propose **major**. Else if any subject starts with `feat:` or `feat(scope):` → propose **minor**. Else → propose **patch**. Then use `AskUserQuestion` to confirm (options: the proposal, the next-larger bump, the next-smaller bump, custom). Pre-1.0 projects (`0.x.y`) often treat minor as breaking — surface that in the question. Don't silently bump.
   - **calver-build** (`YYYY.M.BUILD`, unpadded month): if `YYYY.M` matches today, increment `BUILD`; else reset to `YYYY.M.1`.
   - **calver-month-patch** (`YYYY.MM.PATCH`, zero-padded month): if `YYYY.MM` matches today, increment `PATCH`; else reset to `YYYY.MM.1`.
   - **calver-month** (`YYYY.MM`): use today's `YYYY.MM`; a second release in the same month overwrites.
   - **calver-day** (`YYYY.MM.DD`): use today's date; if it already matches the current version, append/increment a `.N` collision suffix (`…DD` → `…DD.2` → `…DD.3`).
   - **custom**: compute from `version.pattern`'s named groups; if the rule is unclear, ask.

   These are one-line summaries — `references/version-formats.md` holds the authoritative rules and regexes for every format.

2. Update the version file using `Edit`. Use the pattern from config (or auto-detected) to match exactly the line that holds the version, so you change only that line.

3. Verify with a `Grep` after the edit that the new version appears exactly once at the expected location.

## Phase 3 — Simplify and review changes

Invoke the `/simplify` skill via the Skill tool to review all pending changes for reuse, quality, and efficiency. `/simplify` is part of each Claude Code install — assume it's available, no fallback needed.

The skill fans out parallel review agents (reuse, quality, efficiency) over the diff and auto-applies fixes it finds. The subsequent test run (Phase 4) validates that nothing was broken by the simplification. Note which files /simplify modified — you'll surface them at the Phase 7 approval gate so the user's approval covers the shipping code, not just the release notes.

If `/simplify` reports an empty diff, verify with the user — a release with zero pending changes is unusual unless it's a pure docs/changelog release.

**Then, if `.claude/release.config.json` sets `review: true`**, invoke the `/code-review` skill via the Skill tool over the pending diff. `/simplify` does quality cleanups only; `/code-review` is the correctness pass that hunts for bugs. It runs after `/simplify` so the simplification's auto-applied edits get reviewed too. Fix any confirmed correctness finding before continuing. If a confirmed finding can't be resolved, stop the release and surface it — do not ship a known bug. When `review` is absent or `false`, skip this step (projects can also run a review from `.claude/release/preflight.md` instead; see the preflight template).

## Phase 4 — Run tests

Run the detected/configured test command. **Stop immediately if any tests fail.** Do not proceed with a broken build. If the test output is noisy, tail the last failure and present it to the user; the user fixes, then re-invokes the skill.

When you stop here, the version file already holds the new (uncommitted) bump from Phase 2 and /simplify (Phase 3) may have edited source files; both stay in the working tree. On re-invoke, Phase 2's reconciliation step detects the existing bump and offers to resume rather than double-bump — surface this to the user so they know the tree is mid-release.

If config sets `test: "skip"` or test command is `null` and the user explicitly opted in to skipping (via config), document the skip in the release notes. Default behavior is: tests must pass.

## Phase 5 — Review documentation

Check that documentation reflects the current code. The generic checks:
- `README.md` features list matches actual functionality
- Any `docs/*.md` files reference current commands and flags
- New features added since the last release have docs

If `.claude/release/preflight.md` already covered docs, skip the redundant pass — the project's preflight is more specific. Otherwise do the generic pass and fix any drift you find before continuing.

## Phase 6 — Draft release notes

1. Fetch tags from origin (releases tagged on GitHub may not be local):
   ```bash
   git fetch --tags origin 2>/dev/null
   ```
2. Find the previous release tag and assign it to `PREV_TAG`. Read the project's tag prefix from `release.config.json` (`tag.prefix`, default `"v"`). If the prefix is `"v"`:
   ```bash
   PREV_TAG=$(git tag --list 'v*' --sort=-version:refname | head -1)
   ```
   If the prefix is empty, list all tags:
   ```bash
   PREV_TAG=$(git tag --sort=-version:refname | head -1)
   ```
   For any other prefix, substitute it in the `--list` pattern.
3. Collect commits since that tag. **If `PREV_TAG` is empty** — no tags exist yet (a first-ever release) or a prior release shipped without one — use the full history and treat this as the initial release:
   ```bash
   if [ -z "$PREV_TAG" ]; then
     git log --oneline --no-merges          # entire history — initial release
   else
     git log "$PREV_TAG"..HEAD --oneline --no-merges
   fi
   ```
   When `PREV_TAG` is empty, label the draft as the initial release and omit the **Full Changelog** compare line below (there is no previous tag to compare against).
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

## Phase 7 — Approval gate

If /simplify (Phase 3) auto-applied any source edits, first show a `git diff --stat` of the touched files (or a short summary of what changed) so the user sees the code that will ship, not just the notes.

Show the draft notes to the user via `AskUserQuestion` with:
- **Approve** — proceed to commit and release
- **Edit** — user supplies revised notes; loop back here
- **Cancel** — abort the release. Print a summary of the uncommitted changes made so far (the Phase 2 version bump, plus any files /simplify rewrote in Phase 3), leave them in the working tree for manual review, and stop without touching git (no add/commit/tag/push).

Do NOT proceed past this phase without explicit approval.

## Phase 8 — Update CHANGELOG

First check config. If `.claude/release.config.json` sets `changelog.skip: true`, skip this phase entirely (mark the task completed) and continue to Phase 9. Otherwise use `changelog.file` (default `CHANGELOG.md`) as the target file for the steps below.

Insert the approved notes as a new `## <NEW>` section at the top of the changelog file (after the file header, before the previous release section). Use the version *without* the `v` prefix to match conventional changelog entries.

If the changelog file does not exist, ask the user whether to create one — don't silently start one.

## Phase 9 — Splice in pre-commit hook

If `.claude/release/pre-commit.md` exists, Read it and follow its instructions before staging. Common uses: run a formatter, regenerate derived files, run a final lint pass.

Note: pre-commit runs *after* the CHANGELOG entry is added (Phase 8). If the project's pre-commit formatter touches CHANGELOG.md, that's fine — re-stage afterward. If you prefer to format the CHANGELOG before adding the entry, do that work in `preflight.md` instead.

## Phase 10 — Commit, tag, and push

The tag is the shipped artifact — Phase 11's release page, the manual fallback URLs, and the *next* release's Phase 6 previous-tag lookup all depend on it existing. This phase creates and pushes it explicitly; do not rely on `gh release create` (Phase 11) to create it, since that path only runs for GitHub remotes with `gh` installed and never lands the tag locally.

```bash
git status                         # surface what's about to be committed
```

Stage only the files you actually changed during the release: the version file (from Phase 2), `CHANGELOG.md` (from Phase 8), and any files updated by /simplify (Phase 3) or pre-commit hooks (Phase 9). Do NOT use `git add -A` — it risks pulling in untracked files the user didn't author.

```bash
git add <version-file> CHANGELOG.md <files-touched-by-simplify-or-pre-commit>
git commit -m "Release <TAG_PREFIX><NEW>"
git tag -a <TAG_PREFIX><NEW> -m "Release <TAG_PREFIX><NEW>"   # annotated tag — the shipped artifact
git push --follow-tags                                        # pushes the commit and the new tag together
```

Form the tag name with the project's `tag.prefix` (from `.claude/release.config.json`, default `"v"`; empty string means an unprefixed tag). `git push --follow-tags` sends the annotated tag alongside the commit — a plain `git push` would leave the tag local-only.

Show `git status` output before staging. If anything looks unexpected (untracked files you didn't author, paths you don't recognize), pause and ask.

## Phase 11 — Create GitHub release (conditional)

This step depends on what's available. Check both:

1. Run `which gh` to see if the GitHub CLI is installed.
2. Determine whether the remote is GitHub. Parse the URL from `git remote get-url origin` — is the host `github.com`?

**Case A — `gh` is installed AND the remote is github.com:**
```bash
gh release create v<NEW> \
  --title "v<NEW>" \
  --notes "$(cat <<'EOF'
<approved release notes here>
EOF
)"
```
If the project doesn't tag releases with a `v` prefix, use its `tag.prefix` (or follow the existing tag convention). The tag already exists (created and pushed in Phase 10), so `gh release create` attaches the release to it rather than creating a new one. Append `--draft` when `.claude/release.config.json` sets `release.draft: true`, and `--prerelease` when it sets `release.prerelease: true` — without those flags the release publishes live immediately, which defeats a configured draft-for-review workflow.

If `gh release create` exits non-zero (e.g., expired auth), do not treat the release as failed: the commit and tag are already pushed (Phase 10), so the release is shipped — only the release *page* didn't post. Surface the error, tell the user the tag is live, and offer to re-run `gh release create <TAG_PREFIX><NEW> --notes …` (plus any `--draft`/`--prerelease` flags) after they fix `gh auth status`.

**Case B — `gh` is NOT installed, but the remote IS github.com:**
The release is already shipped (the tag was pushed in Phase 10). Print manual instructions for the release page:

- URL to open: `https://github.com/<owner>/<repo>/releases/new?tag=v<NEW>`
- Paste the approved release notes (from Phase 6) into the description field.
- Click "Publish release."

Then continue to Phase 12. Do not abort the workflow — the tag is the shipped artifact; the release page is documentation.

**Case C — the remote is NOT github.com** (GitLab, Gitea, Codeberg, self-hosted, etc.):
GitHub release pages don't apply. The release is shipped via the tag pushed in Phase 10. If the host has its own equivalent:
- GitLab: `https://gitlab.com/<owner>/<repo>/-/releases/new?tag_name=v<NEW>`
- Gitea/Codeberg: `<host>/<owner>/<repo>/releases/new?tag=v<NEW>`

Point the user at the right URL if you can recognize the host; otherwise just confirm the tag is pushed and let them handle host-specific release pages manually.

## Phase 12 — Splice in post-release hook

If `.claude/release/post-release.md` exists, Read it and follow its instructions. Common uses: deploy docs, post to a channel, trigger a deploy pipeline, publish to a package registry.

## Phase 13 — Confirm completion

Print a one-line summary:
```
released v<NEW>: <github release url>
```

## Important rules

- Never skip tests silently. If tests fail, stop.
- Never commit or push before the user approves the release notes.
- Never bypass the user-defined preflight if it exists — it encodes project invariants.
- Never ship past a confirmed `/claude-security` finding when `security: true` — resolve it or stop the release.
- Never use `git add -A` for the release commit — stage specific files (version file, CHANGELOG, simplify/pre-commit changes). Always show `git status` first.
- If the user's project has unusual conventions you didn't account for, ASK — don't guess.
- If two version sources disagree, ASK — don't pick.
- Treat `.claude/release/*.md` files as authoritative project guidance. They override generic behavior in their phase.

## See also

- `references/version-formats.md` — version detection priority list, format presets, regex patterns
- `references/config-schema.md` — full `release.config.json` schema with examples
- `references/lifecycle.md` — which lifecycle markdown gets spliced when, and how to write one
