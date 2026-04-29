# Lifecycle Markdown Files

The `/claude-release:release` skill checks for four optional markdown files in a project's `.claude/release/` directory and splices their content into the release flow at specific phases. Each file is read and followed *as instructions for Claude*, not as a script — they can mix prose reasoning, bash blocks, checklists, and conditionals.

## The four slots

| File | Splice phase | When it runs | Common uses |
|---|---|---|---|
| `preflight.md` | 1 | After detection, before version bump (Phase 2) | Project integrity checks, custom doc audits, sanity tests |
| `pre-commit.md` | 8 | After CHANGELOG update (Phase 7), before staging and `git commit` (Phase 9) | Format/lint passes, regenerate derived files |
| `post-release.md` | 11 | After `gh release create` succeeds (Phase 10) | Deploy docs, post to a channel, trigger pipelines, publish to a registry |
| `notes-template.md` | 5 | While drafting release notes, before the approval gate (Phase 6) | Required sections, custom headings, content callouts |

If a file is absent, the skill skips that splice silently. There is no need to declare which slots a project uses.

## How splicing works

When the skill reaches the relevant phase, it:
1. Checks `.claude/release/<slot>.md` exists.
2. If yes: reads the file with `Read` and treats its contents as authoritative project guidance for that phase.
3. Project guidance can extend, restrict, or replace the generic behavior — whichever the markdown specifies.

The user-defined markdown wins on conflicts within its phase. The generic flow's invariants (test failures abort, approval gate is required, no commit without approval) are NOT overridable — they live in the skill body, not the splice.

## Writing a lifecycle markdown

Lifecycle markdowns are read by Claude as instructions, not executed as scripts. Claude reads the file, runs the bash blocks via its Bash tool, and decides what to do based on the prose around them. Write them like you'd write any instruction:

- Be explicit about what to check, fix, or run.
- Use bash blocks for commands you want Claude to execute.
- Use prose for reasoning ("if X, do Y; otherwise Z").
- If the check produces output you want Claude to inspect, say so.
- **To block the release, say so in prose**: "stop the release", "abort", "do not continue". Don't rely on `exit 1` inside a bash block — Claude reads the markdown, it doesn't `bash` the file, so a bare `exit 1` is just text. The skill's "stop on failure" rule kicks in based on what Claude infers from the output and your prose.

### Example: `preflight.md` for claude-sessions
```markdown
# Preflight: install/uninstall parity

The cs tool ships hooks, binaries, and settings via `install.sh`. The
uninstall logic in `bin/cs::run_uninstall()` must mirror it exactly,
or users will leave artifacts behind.

Verify all three sources agree:

- All `hooks/*.sh` files are installed by `install.sh`.
- Every binary copied by `install.sh` is removed by `run_uninstall`.
- Every settings.json hook event added by `install.sh` is cleaned up.

Run:

\`\`\`bash
REPO=$(ls hooks/*.sh | xargs -I{} basename {} | sort)
INSTALLED=$(rg -o 'cp "\$HOOKS_SOURCE/[^"]+' install.sh | sed 's/.*\///' | sort)
UNINSTALLED=$(rg "for hook in" bin/cs | grep -oE '[a-z-]+\.sh' | sort)
diff <(echo "$REPO") <(echo "$INSTALLED")
diff <(echo "$REPO") <(echo "$UNINSTALLED")
\`\`\`

If the diffs are non-empty, fix all three locations (install.sh,
run_uninstall in bin/cs, and docs/hooks.md) before continuing. Do not
release with drift.
```

### Example: `pre-commit.md` for a TypeScript project
```markdown
# Pre-commit: format and typecheck

Before committing the release, run:

\`\`\`bash
pnpm format
pnpm typecheck
\`\`\`

Re-stage any files the formatter changed. If typecheck fails, stop
the release — type errors must be fixed before shipping.
```

### Example: `post-release.md` for a docs site
```markdown
# Post-release: deploy docs

The release has been published. Now deploy the docs site:

\`\`\`bash
pnpm build:docs
pnpm deploy:docs
\`\`\`

Verify the deploy succeeded by curling the docs URL and checking the
version string matches the release.
```

### Example: `notes-template.md`
```markdown
# Release notes template

Use this structure for the draft notes:

## What's Changed

### Highlights
[1-2 sentence summary of the most important change]

### Features
[grouped commits]

### Fixes
[grouped commits]

### Breaking Changes
[anything that requires user action; list migration steps]

### Internal
[refactors, deps, build]

**Full Changelog**: <compare link>

---

If there are no breaking changes, omit that section entirely.
If there are, lead with them — users need to see them first.
```

## When NOT to use lifecycle markdowns

- For deterministic checks that don't need reasoning (a single lint command), use `preflightCmd` in `release.config.json` instead. Lighter, faster, no Claude invocation.
- For documentation about how to release (audience: humans), use `docs/release.md` in your project. The lifecycle slots are for *runnable* instructions, not reference material.
- For one-off release notes (not a recurring template), just edit the draft when the approval gate appears. Don't add a `notes-template.md` for a single release.
