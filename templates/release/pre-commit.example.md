# Pre-commit hook

Runs in Phase 8, after the CHANGELOG entry is added and before `git commit`.
Use this for format/lint passes or to regenerate derived files that must
be in the release commit.

## Example: format and re-stage
```bash
pnpm format
git add -u
```

## Example: regenerate API docs
```bash
pnpm gen:api-docs
git add docs/api/
```

## Example: verify build still works
```bash
pnpm build
```
If this fails, stop the release. A version that doesn't build should not ship.

Keep this fast — it runs every release. If a check is slow and only matters
occasionally, move it to `preflight.md`.
