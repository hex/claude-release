# Preflight checks

This file runs in Phase 1 of `/claude-release:release`, after project detection
and before the version bump. Use it for project-specific integrity checks
that the generic flow doesn't know about.

Claude reads this file and follows it as instructions — it runs the bash
blocks and reasons about the output. To block the release, say so clearly
in prose; the skill stops based on what Claude infers, not on shell exit
codes inside these blocks.

Replace this content with checks for your project. Examples:

## Example: verify lockfile is committed

Run this and check the output:

```bash
git status --porcelain pnpm-lock.yaml
```

If the output is non-empty, the lockfile has uncommitted changes. **Stop the
release** and tell the user to commit the lockfile first.

## Example: verify generated files are up to date

```bash
pnpm gen:types
git status --porcelain types/
```

If the output is non-empty, generated types were stale. **Stop the release**
and tell the user to commit the regenerated files.

## Example: correctness review of the pending diff

Invoke the `/code-review` skill on the pending working-tree changes. If it
reports confirmed correctness findings, fix them; if a finding can't be
resolved, **stop the release** and surface it.

Note: preflight runs before the version bump and `/simplify`, so a review
here won't see the edits `/simplify` auto-applies in Phase 3. If you want
those reviewed too, prefer `"review": true` in `release.config.json`, which
runs `/code-review` in Phase 3 after `/simplify`.

## Example: security scan of a specific area

Set `"security": true` in `release.config.json` for the common case — it runs
`/claude-security` in this same phase, scoped to the changes the release
ships. Write a scan here instead when you want a *different* scope every
release, e.g. always scanning the same sensitive area regardless of what
changed:

Invoke `/claude-security` and ask it to scan `src/auth/` in full. If the
report lists confirmed findings, resolve them (its Suggest patches flow
builds the fix); if one can't be resolved, **stop the release**.

A full-area scan is slower and costs more tokens than the change scan the
config flag runs — only worth it for code where a missed vulnerability is
expensive. Don't enable both for the same area; you'd pay twice.

## Example: docs cross-check

Verify the README's command list matches `bin/<your-cli> --help` output. If
they drift, update the README before continuing — that's not a stop, just a
fix-and-proceed.

## When to stop vs fix-and-proceed

- **Stop the release** for: dirty working tree on critical files, version
  source disagreement, known broken state. Tell the user explicitly.
- **Fix and proceed** for: doc drift, formatting, regenerable files where the
  fix is obvious and reversible.
