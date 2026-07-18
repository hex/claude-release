# Changelog

All notable changes to claude-release are documented here. The format follows the categorization used by `/claude-release:release` itself: Features, Fixes, Docs, Other.

<!-- /claude-release:release inserts new versions here, above older ones. -->

## 2026.7.2

### Features
- Optional `/code-review` correctness pass in the release flow. Set `"review": true` in `.claude/release.config.json` and Phase 3 — now "Simplify and review changes" — runs `/code-review` over the pending diff after `/simplify`, so auto-applied simplifications get correctness-checked too. Confirmed findings must be fixed before the release continues; unresolvable ones stop it. Off by default.
- The preflight template gains a `/code-review` example for reviewing before the version bump, with a note that the config flag is preferred (preflight-time review misses `/simplify`'s edits).

### Other
- Tests skipped per `test: "skip"` config (markdown-only plugin; no test suite).

## 2026.7.1

First tagged release of the `claude-release` plugin.

### Features
- Project-agnostic `/claude-release:release` command driving the full 14-phase flow
- Auto-detection of version file, format, and test command across Node, Rust, Python, Go, Ruby, Shell, PHP, and .NET
- Per-project `.claude/release.config.json` (version source and format, test command, tag prefix, changelog path, draft/prerelease, repo override)
- Lifecycle markdown hooks: `preflight.md`, `pre-commit.md`, `post-release.md`, `notes-template.md`
- `/simplify` integration to review pending changes before they ship
- Conditional GitHub release via `gh`, with a manual-URL fallback for no-`gh` and non-GitHub hosts (GitLab, Gitea, Codeberg)

### Fixes
- Phase 10 now creates and pushes an annotated tag (`git tag -a` plus `git push --follow-tags`); previously no tag was created, so releases on non-GitHub hosts shipped tagless
- Reconcile an uncommitted prior version bump before bumping, preventing a silent double-bump when a release is re-invoked after a test failure
- Honor documented config that was previously ignored: `release.draft`/`prerelease`, `changelog.file`/`skip`, `repo.owner`/`name`
- First-release notes fallback when no prior tag exists, plus explicit `PREV_TAG` assignment
- Approval gate gains a Cancel option; `/simplify`'s auto-applied edits are surfaced before the push
- Phase 2 version-format digest lists all five calver presets (fixes calver-day being folded into calver-month)

### Docs
- The repo now ships its own `.claude/release.config.json`, so the plugin is self-releasable and dogfoods the config feature
- README auto-detection table adds PHP and .NET; troubleshooting reworded off literal error strings; skill and README synced with the corrected flow
