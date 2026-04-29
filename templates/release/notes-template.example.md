# Release notes template

Used in Phase 5 of `/claude-release:release` when drafting release notes.
Defines the structure, sections, and any required content callouts.

## Template

```markdown
## What's Changed

### Highlights
[1-2 sentence summary of the most important user-visible change]

### Features
[group commits prefixed with feat: or matching pattern "Add X"]

### Fixes
[group commits prefixed with fix: or matching pattern "Fix X"]

### Breaking Changes
[anything that requires user action; include migration steps]

### Internal
[refactors, dependency bumps, build/CI changes — keep brief]

**Full Changelog**: <compare link>
```

## Rules

- Lead with **Breaking Changes** if any exist — users need to see them first.
- Omit empty sections (no "Features: none").
- If commits are uncategorized, ask the user before forcing them into a section.
- Link to migration guides for breaking changes.
- Use plain language; avoid internal jargon in user-facing notes.
