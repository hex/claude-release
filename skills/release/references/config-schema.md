# `.claude/release.config.json` Schema

Per-project override for the claude-release plugin. All fields are optional; values not present fall back to auto-detection (see `version-formats.md`).

## Schema

```json
{
  "version": {
    "file": "string (path, e.g., 'bin/cs' or 'package.json')",
    "format": "semver | calver-build | calver-month-patch | calver-month | calver-day | custom",
    "pattern": "string (regex with named groups, only when format=custom)"
  },
  "test": "string (shell command) | 'skip'",
  "review": "boolean (default: false; run /code-review on the pending diff in Phase 3, after /simplify)",
  "security": "boolean (default: false; run /claude-security over the release's committed changes in Phase 1)",
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
- `calver-build` — `YYYY.M.BUILD` (unpadded month, build resets each month). Used by claude-sessions and claude-release itself.
- `calver-month-patch` — `YYYY.MM.PATCH` (zero-padded month, patch resets each month).
- `calver-month` — `YYYY.MM`.
- `calver-day` — `YYYY.MM.DD`.
- `custom` — supply your own `pattern`.

### `version.pattern`
Regex with named capture groups (`year`, `month`, `build`, `major`, `minor`, `patch`, `prerelease`). Required when `format=custom`. The skill computes the next version using whichever groups are present; if rules are unclear, it asks.

### `test`
Shell command for the test step. Use `"skip"` to explicitly opt out (rare; documented in release notes when used).

### `review`
When `true`, Phase 3 runs the `/code-review` skill over the pending diff after `/simplify`. `/simplify` covers quality (reuse, simplification, efficiency); `/code-review` covers correctness. Confirmed findings must be fixed before the release continues; unresolvable ones stop it. Off by default because it adds latency and token cost to every release. To review *before* the version bump instead, run `/code-review` from `.claude/release/preflight.md` — but note that path misses the edits `/simplify` applies later.

### `security`
When `true`, Phase 1 runs the `/claude-security` plugin over the changes the release ships — a multi-agent vulnerability scan whose findings are published only after independent verifier agents confirm them. Confirmed findings must be resolved before the release continues; unresolvable ones stop it.

Three things distinguish it from `review`:

- **It runs in Phase 1, not Phase 3.** `/claude-security` change scans read *committed* code only, and from Phase 2 onward the working tree is deliberately dirty (the version bump, then `/simplify`'s edits). Phase 1 is the last point where the tree matches a commit. The trade is the same one preflight-time `/code-review` makes: the scan won't see the edits `/simplify` applies later, and won't see uncommitted work already in the tree. The skill states that gap out loud rather than implying full coverage.
- **It needs a plugin.** `/simplify` and `/code-review` ship with every Claude Code; `/claude-security` is installed from the official marketplace (`/plugin install claude-security@claude-plugins-official`). If it's missing, the skill asks whether to install and re-run or continue without the scan — it never skips the gate silently.
- **It costs more.** A scan is a multi-agent run that can take a while and use a significant number of tokens. On a large repository, scope it: the plugin offers focused areas, and its report states what was and wasn't examined.

Results land in a timestamped `CLAUDE-SECURITY-<timestamp>/` directory that carries its own `.gitignore`, so it stays out of the release commit.

Off by default. `review` and `security` are independent — enable either, both, or neither.

### `preflightCmd`
A single shell command run during Phase 1 *in addition to* `.claude/release/preflight.md` and the `security` scan (Phase 1 runs whichever of the three are configured, cheapest first). Use this for purely deterministic checks (lint, format-check) that don't need Claude to reason about output. Non-zero exit aborts the release.

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

### Both review gates enabled
```json
{
  "review": true,
  "security": true
}
```
Phase 1 scans the release's committed changes for vulnerabilities; Phase 3 correctness-reviews the pending diff after `/simplify`. Adds real latency and token cost to every release — worth it for code where a shipped bug or vulnerability is expensive, overkill for a docs repo.
