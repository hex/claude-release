# Post-release tasks

Runs in Phase 11, after `gh release create` succeeds. The release is
already public at this point — don't put anything here that could fail
the release if it errors. Use this for follow-up tasks: deploys,
announcements, registry pushes.

## Example: publish to npm
```bash
npm publish
```

## Example: deploy docs site
```bash
pnpm build:docs
pnpm deploy:docs
```

## Example: trigger a CI deploy pipeline
```bash
gh workflow run deploy.yml --ref "v$NEW_VERSION"
```

## Example: post a release announcement
```bash
gh issue create \
  --title "Released v$NEW_VERSION" \
  --body "See https://github.com/$OWNER/$REPO/releases/tag/v$NEW_VERSION" \
  --label announcement
```

## Failure handling

If a post-release step fails, the release is already out — surface the
error to the user and let them decide whether to retry, work around it,
or accept the partial state. Do NOT try to "undo" the release.
