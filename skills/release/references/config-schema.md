# `.claude/release.config.json` Schema

Per-project override for the release-flow plugin. All fields are optional; values not present fall back to auto-detection (see `version-formats.md`).

## Schema

```json
{
  "version": {
    "file": "string (path, e.g., 'bin/cs' or 'package.json')",
    "format": "semver | calver-build | calver-month-patch | calver-month | calver-day | custom",
    "pattern": "string (regex with named groups, only when format=custom)"
  },
  "test": "string (shell command) | 'skip'",
  "preflightCmd": "string (shell command run before any phase begins)",
  "repo": {
    "owner": "string (overrides git remote parse)",
    "name": "string (overrides git remote parse)"
  },
  "tag": {
    "prefix": "string (default: 'v'; set to '' for unprefixed tags)"
  },
  "changelog": {
    "file": "string (default: 'CHANGELOG.md')",
    "skip": "boolean (default: false)"
  },
  "release": {
    "draft": "boolean (default: false; pass --draft to gh release create)",
    "prerelease": "boolean (default: false; pass --prerelease to gh release create)"
  }
}
```

## Field reference

### `version.file`
Path to the file holding the canonical version. Required only if multiple files disagree or auto-detection picks the wrong one.

### `version.format`
One of:
- `semver` — `MAJOR.MINOR.PATCH`. Default for most projects.
- `calver-build` — `YYYY.M.BUILD` (unpadded month, build resets each month). Used by claude-sessions and release-flow itself.
- `calver-month-patch` — `YYYY.MM.PATCH` (zero-padded month, patch resets each month).
- `calver-month` — `YYYY.MM`.
- `calver-day` — `YYYY.MM.DD`.
- `custom` — supply your own `pattern`.

### `version.pattern`
Regex with named capture groups (`year`, `month`, `build`, `major`, `minor`, `patch`, `prerelease`). Required when `format=custom`. The skill computes the next version using whichever groups are present; if rules are unclear, it asks.

### `test`
Shell command for the test step. Use `"skip"` to explicitly opt out (rare; documented in release notes when used).

### `preflightCmd`
A single shell command run during Phase 1 *in addition to* `.claude/release/preflight.md` (if both exist). Use this for purely deterministic checks (lint, format-check) that don't need Claude to reason about output. Non-zero exit aborts the release.

### `repo.owner` / `repo.name`
Override the GitHub repo derived from `git remote get-url origin`. Useful for monorepos where the release page lives in a different repo, or for repos with non-standard remote URLs.

### `tag.prefix`
String prepended to the version when tagging. Default `"v"` produces `v1.2.3`. Set `""` for unprefixed tags. This also affects the GitHub release name.

### `changelog.file`
Path to the changelog. Default `CHANGELOG.md`. Set `changelog.skip: true` to disable changelog updates entirely (only do this if your project genuinely doesn't maintain one).

### `release.draft` / `release.prerelease`
Pass through to `gh release create`. Useful for staged releases.

## Examples

### Minimal — Node project (no config needed)
No `release.config.json`. Auto-detection finds `package.json`, infers `semver`, runs `npm test`, parses `git remote` for repo. The plugin Just Works.

### claude-sessions (calver with shell-script version file)
```json
{
  "version": {
    "file": "bin/cs",
    "format": "calver-build"
  },
  "test": "bash tests/test_*.sh",
  "tag": {
    "prefix": "v"
  }
}
```

### Polyglot monorepo (Rust + Python, version in Rust crate)
```json
{
  "version": {
    "file": "core/Cargo.toml",
    "format": "semver"
  },
  "test": "cargo test --workspace && pytest python/",
  "preflightCmd": "cargo fmt --check && ruff check"
}
```

### Project with custom calver
```json
{
  "version": {
    "file": "VERSION",
    "format": "custom",
    "pattern": "^(?P<year>\\d{4})\\.(?P<quarter>Q[1-4])\\.(?P<patch>\\d+)$"
  }
}
```

### Draft releases for review
```json
{
  "release": {
    "draft": true
  }
}
```
Releases are created as drafts; the user reviews on GitHub before publishing.
