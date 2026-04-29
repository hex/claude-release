# claude-release

A Claude Code plugin that turns "ship a release" into a single slash command. Auto-detects most projects; extensible for the rest.

## What it does

`/claude-release:release` drives the full release workflow:

1. **Detect** version file, format, test command, and git remote
2. **Preflight** (project-specific checks, optional)
3. **Bump** the version
4. **Simplify** pending changes — invokes `/simplify` to catch dupes/inefficiencies before they ship
5. **Test** — abort on failure (validates simplifications too)
6. **Review docs** for drift
7. **Draft release notes** grouped by Features / Fixes / Docs / Other
8. **Approval gate** — you confirm or edit the notes before anything ships
9. **Update CHANGELOG.md**
10. **Pre-commit hook** (project-specific, optional)
11. **Commit, tag, push**
12. **Create release page** — auto via `gh` if installed and the remote is GitHub; otherwise emits a manual URL (works on GitLab, Gitea, self-hosted git too)
13. **Post-release hook** (project-specific, optional)
14. **Confirm** — print the release URL

## Installation

### From marketplace (recommended)

```
# Add the hex-plugins marketplace (once)
/plugin marketplace add hex/claude-marketplace

# Install the plugin
/plugin install claude-release
```

### From GitHub

```
/plugin install hex/claude-release
```

### Manual

```bash
git clone https://github.com/hex/claude-release.git
claude --plugin-dir /path/to/claude-release
```

After installation, `/claude-release:release` is available as a slash command. Run `/help` to confirm it appears.

## Prerequisites

- Git remote `origin` set
- Either a recognizable version file (see below) or a `.claude/release.config.json`
- *(Optional)* `gh` CLI authenticated. If `gh` is installed and the remote is `github.com`, `/claude-release:release` auto-creates the GitHub release page. Without `gh`, the release still ships (tag is pushed) — you'll get a manual URL to paste the release notes into. Non-GitHub hosts (GitLab, Gitea, etc.) get the same fallback with a host-appropriate URL.

## Auto-detection

For most projects, no configuration is needed. The plugin detects:

| Project type | Version source | Test command |
|---|---|---|
| Node | `package.json` `"version"` | `npm test` (or `pnpm test` / `yarn test`) |
| Rust | `Cargo.toml` `[package].version` | `cargo test` |
| Python | `pyproject.toml` `[project].version` | `pytest` |
| Go | tags only (use config) | `go test ./...` |
| Ruby | `*.gemspec` `spec.version` | `bundle exec rspec` |
| Shell | `bin/<script>` `VERSION=` line | `bash tests/test_*.sh` if present |
| Generic | `VERSION` file at root | ask if not detected |

If you have multiple version sources that disagree, the plugin asks you which is canonical — it won't guess.

## Per-project configuration

Drop a `.claude/release.config.json` in your project to override detection:

```json
{
  "version": {
    "file": "bin/cs",
    "format": "calver-build"
  },
  "test": "bash tests/test_*.sh",
  "tag": { "prefix": "v" }
}
```

Full schema: see `skills/release/references/config-schema.md` after install, or `templates/release.config.example.json` for a starter.

## Per-project lifecycle hooks (optional)

For things that need *Claude reasoning* — checks with edge cases, format conversions, conditional logic — drop markdown files in `.claude/release/`:

| File | Phase | Purpose |
|---|---|---|
| `preflight.md` | Before version bump | Integrity checks, custom audits |
| `pre-commit.md` | Before `git commit` | Format/lint, regenerate derived files |
| `post-release.md` | After GitHub release | Deploy, announce, publish to registry |
| `notes-template.md` | While drafting notes | Required sections, custom structure |

Starters live in `templates/release/`. Copy what you need:

```bash
cp claude-release/templates/release/preflight.example.md \
   .claude/release/preflight.md
```

The plugin reads these files and follows their instructions when it reaches that phase. They override the generic flow within their phase. The skill's invariants (test failures abort, approval is required, no commit without approval) are not overridable.

## Quickstart

```bash
# In any project with a recognizable version file:
/claude-release:release

# Or pin a specific version:
/claude-release:release 2.1.0
```

Walk through the prompts. The plugin will pause for your approval before anything is committed or pushed.

## How extension precedence works

For each phase that has a splice point:

1. Plugin runs the generic logic for the phase.
2. If `.claude/release/<slot>.md` exists, the plugin reads it and follows its instructions — they can extend, restrict, or replace the generic behavior for that phase.
3. Skill invariants (approval required, tests must pass) are NOT overridable; they live in the skill body.

For deterministic checks that don't need reasoning, prefer `release.config.json` `preflightCmd` (a single shell command) over a `preflight.md`. The markdown is for things where Claude needs to interpret output and decide.

## Troubleshooting

**"Multiple version sources detected"** — you have, e.g., both `package.json` and `Cargo.toml`. Add `version.file` to your config to disambiguate.

**"No test command detected"** — auto-detection couldn't find one. Set `test` in your config, or add it to your project (e.g., a `test` script in `package.json`).

**"gh release create failed"** — check `gh auth status`. The plugin won't roll back commits/tags on a release failure; the tag is already pushed, so just fix the auth issue and re-run `gh release create v<NEW> --notes ...` manually with the approved notes.

**"I don't have `gh` installed"** — that's fine. The release still ships (Phase 11 pushes the tag); the plugin will print a manual URL like `https://github.com/<owner>/<repo>/releases/new?tag=v<NEW>` for you to open and paste the approved release notes into. Same fallback works for GitLab, Gitea, and Codeberg with host-appropriate URLs.

**"My preflight says fix the docs but I don't agree"** — preflight is project-defined. Edit `.claude/release/preflight.md` to match your actual project invariants.

## License

MIT — see [LICENSE](LICENSE).
