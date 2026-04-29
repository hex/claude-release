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

## Example: docs cross-check

Verify the README's command list matches `bin/<your-cli> --help` output. If
they drift, update the README before continuing — that's not a stop, just a
fix-and-proceed.

## When to stop vs fix-and-proceed

- **Stop the release** for: dirty working tree on critical files, version
  source disagreement, known broken state. Tell the user explicitly.
- **Fix and proceed** for: doc drift, formatting, regenerable files where the
  fix is obvious and reversible.
