# Version Formats and Detection

This file describes how `/release-flow:release` discovers the version file, format, and test command for a project. Read this when Phase 0 of `SKILL.md` runs.

## Detection priority (version source)

Walk the list top-to-bottom. The first source that yields a version is canonical, **unless** another source later in the list also yields a version *and disagrees* — in that case, use `AskUserQuestion` to ask which is canonical.

1. **`.claude/release.config.json`** — explicit override. Highest priority.
2. **`package.json`** — `"version"` top-level field. Format: semver.
3. **`pyproject.toml`** — `[project] version` (PEP 621) or `[tool.poetry] version`. Format: PEP 440 (semver-compatible).
4. **`Cargo.toml`** — `[package] version`. Format: semver.
5. **`go.mod`** — module version typically in tags only; skip unless config points here.
6. **`composer.json`** — `"version"` field (often absent; tags are canonical for Composer).
7. **`*.gemspec`** — `spec.version = "X.Y.Z"`. Format: semver.
8. **`VERSION`** file at repo root — single line, just the version.
9. **`bin/<script>`** with a `VERSION=` line — shell scripts that embed their version (e.g., claude-sessions' `bin/cs`).
10. **`*.csproj` / `*.fsproj`** — `<Version>` element. Format: semver.

If none match: `AskUserQuestion` asks the user where the version lives.

## Format presets

A format determines how to compute the next version when no override is given.

### `semver` (default)
Pattern: `MAJOR.MINOR.PATCH` (with optional `-prerelease+build`).
Bump rule: inspect commits since the previous tag and propose using Conventional Commits semantics, then confirm via `AskUserQuestion`.
- `BREAKING CHANGE:` in commit body OR `!:` after type (e.g., `feat!:`, `fix!:`) → propose **major** bump (`X.0.0`).
- Subject starts with `feat:` or `feat(scope):` → propose **minor** bump (`x.Y.0`).
- Otherwise (`fix:`, `docs:`, `chore:`, `refactor:`, etc.) → propose **patch** bump (`x.y.Z`).
Pre-1.0 projects (`0.x.y`) often treat minor as breaking — flag this in the AskUserQuestion when on a `0.x.y` version. The proposal sets the default; the question is the gate.
Regex: `^(\d+)\.(\d+)\.(\d+)(?:-[\w\.\-]+)?(?:\+[\w\.\-]+)?$`

### `calver-build`
Pattern: `YYYY.M.BUILD` where `M` has no leading zero and `BUILD` resets each month.
Bump rule:
- If current `YYYY.M` matches today: `BUILD += 1`.
- Otherwise: reset to today's `YYYY.M.1`.
Regex: `^(\d{4})\.(\d{1,2})\.(\d+)$`
Example: `2026.4.42` → `2026.4.43` (same month) or `2026.5.1` (next month).
Used by: this plugin (`release-flow`) itself, and by claude-sessions.

### `calver-month-patch`
Pattern: `YYYY.MM.PATCH` — same logical layout as `calver-build` but with **zero-padded** month. `PATCH` resets each month.
Bump rule:
- If current `YYYY.MM` matches today (today's month zero-padded): `PATCH += 1`.
- Otherwise: reset to today's `YYYY.MM.1`.
Regex: `^(\d{4})\.(\d{2})\.(\d+)$`
Example: `2026.04.1` → `2026.04.2` (same month) or `2026.05.1` (next month).

### `calver-month`
Pattern: `YYYY.MM` (zero-padded month).
Bump rule: use today's `YYYY.MM`. Multiple releases in one month overwrite.
Regex: `^(\d{4})\.(\d{2})$`

### `calver-day`
Pattern: `YYYY.MM.DD` (with optional `.N` suffix for same-day collisions).
Bump rule: today's date. If today's `YYYY.MM.DD` already matches the current version, append `.2`; if it ends in `.N`, increment `N`.
Regex (no suffix): `^(\d{4})\.(\d{2})\.(\d{2})$`
Regex (with collision suffix): `^(\d{4})\.(\d{2})\.(\d{2})\.(\d+)$`
Examples: `2026.04.29` (first release that day) → `2026.04.29.2` (second) → `2026.04.29.3` (third) → `2026.04.30` (next day, suffix resets).

### `custom`
Set `version.pattern` in config to a regex with named capture groups (`year`, `month`, `build`, `major`, `minor`, `patch`). The skill computes the next value by your rules — when in doubt, ask.

## Detection priority (test command)

1. **`.claude/release.config.json`** `test` field — explicit override.
2. **`package.json`** `scripts.test` → `npm test` (or `pnpm test` / `yarn test` if a lockfile is present for that manager).
3. **`Cargo.toml`** present → `cargo test`.
4. **`pyproject.toml`** with `[tool.pytest]` or `tests/` directory → `pytest` (prefer `uv run pytest` if `uv.lock` is present).
5. **`go.mod`** present → `go test ./...`.
6. **`Gemfile`** with `rspec` or `spec/` directory → `bundle exec rspec`; with `rake` test task → `bundle exec rake test`.
7. **`tests/test_*.sh`** present → `bash tests/test_*.sh` (claude-sessions pattern).
8. **`Makefile`** with `test:` target → `make test`.
9. **None matched** → `AskUserQuestion` asks the user. Do NOT skip tests silently.

## Edit-the-version-line patterns

To bump the version safely (one-line `Edit`), use these patterns to match exactly the line to change. The skill should match the line, then replace just the version literal.

| File | Match pattern (regex) | Replace just |
|---|---|---|
| `package.json` | `"version"\s*:\s*"([^"]+)"` | the captured version literal |
| `Cargo.toml` | `^version\s*=\s*"([^"]+)"` (in `[package]` section) | the captured version literal |
| `pyproject.toml` | `^version\s*=\s*"([^"]+)"` (in `[project]` or `[tool.poetry]`) | the captured version literal |
| `VERSION` | entire file content | replace whole line |
| `bin/<script>` | `^VERSION="([^"]+)"` or `^VERSION=([0-9a-zA-Z\.\-]+)` | the captured version literal |
| `*.gemspec` | `\.version\s*=\s*"([^"]+)"` | the captured version literal |

After the edit, run a `Grep` to verify the new version appears exactly once at the expected location.

## Conflict resolution

When two detected sources disagree (e.g., `package.json` says `1.2.3` but `Cargo.toml` says `1.2.4`):

1. List both sources and their values.
2. `AskUserQuestion` with options:
   - "Use `<source A>` as canonical and update `<source B>` to match"
   - "Use `<source B>` as canonical and update `<source A>` to match"
   - "Update both manually before continuing" (abort the release)
3. Whichever is chosen, edit BOTH files in Phase 2 so they stay in sync.

For polyglot projects, recommend the user write `.claude/release.config.json` with an explicit `version.file` to avoid this question every release.
