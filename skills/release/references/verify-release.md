# Verify release (Phase 13)

Answers one question: **is `<TAG>` actually released?** The check reads live state (the remote, the GitHub API, the package registry, a fresh install) and never the local checkout, the release notes, or what earlier phases reported. Adapted from openclaw's `verify-release` skill (MIT).

## Ground rules

- **Read only.** Nothing in this phase publishes, tags, pushes, edits a release, or re-runs a registry publish. If a surface fails, report it and let the user decide on a fix.
- **Never print secrets.** Don't echo tokens, `.npmrc` contents, or auth headers. `gh auth status` is fine; it masks its token line.
- **Only the surfaces the project ships.** Derive them from Phase 0's detection. List a surface the project doesn't ship under caveats as intentionally absent; never skip it silently.
- **Real `HOME` for everything except the install smoke.** `git ls-remote`, `gh`, `npm view` need the user's credentials. Only surface 5 runs under an isolated `HOME`.

Below, `<TAG>` is `<TAG_PREFIX><NEW>` and `<V>` is `<NEW>`.

## 1. Tag

```bash
git ls-remote --tags origin "refs/tags/<TAG>" "refs/tags/<TAG>^{}"
git rev-parse "<TAG>^{commit}"
```

For an annotated tag, the `^{}` line is the commit. It must equal the local `<TAG>^{commit}` from Phase 10. No output from `ls-remote` means the tag never reached origin: the answer is **no**.

## 2. Release page

Only when Phase 11 was Case A or B (GitHub remote).

```bash
gh release view "<TAG>" --json tagName,isDraft,isPrerelease,targetCommitish,publishedAt,assets,url
```

- `isDraft` must equal `release.draft` from config (default `false`). A draft that config asked for is correct, but the answer to "is it released?" is still **no, draft awaiting publish**.
- `isPrerelease` must equal `release.prerelease` (default `false`).
- A published release carries a `publishedAt` timestamp.
- `assets`: report the count and names. The flow uploads none itself, so expect zero unless `post-release.md` uploads some; if it does, every asset it names must appear.

Non-GitHub hosts (Case C): the tag is the release. List the release page as not checked under caveats.

## 3. Version files at the tag

Read from the tag object, never the working tree, so a dirty checkout doesn't matter:

```bash
git show "<TAG>:<version-file>"
```

Do this for the configured `version.file` and for every other version source Phase 0 detected (`package.json`, `pyproject.toml`, `Cargo.toml`, `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`, and so on). Each must say `<V>`.

**If the local tag is missing or its commit differs from surface 1's remote commit**, the checkout has diverged. Don't read the local tag. Fetch the tarball of the remote tag into a temp dir and read the files there:

```bash
VERIFY_SRC=$(mktemp -d)
gh api "repos/<owner>/<repo>/tarball/<TAG>" | tar -xz -C "$VERIFY_SRC" --strip-components=1
```

Record under caveats that the checkout diverged and that versions came from the tarball.

## 4. Registry readback

Only for registries the project publishes to, which in this flow means `post-release.md` publishes there.

| Ecosystem | Command | Must hold |
|---|---|---|
| npm | `npm view <pkg>@<V> version dist-tags --json` | `version` is `<V>`; for a stable release `dist-tags.latest` is `<V>` |
| PyPI | `curl -fsS https://pypi.org/pypi/<pkg>/<V>/json` | HTTP 200 and `info.version` is `<V>` |
| crates.io | `cargo search <crate> --limit 1` | the listed version is `<V>` |
| Claude Code plugin | the marketplace's `marketplace.json` on its default branch | read the paragraph after this table |

Registries can lag a publish by a minute or two, and npm mirrors longer. If the readback misses, wait about 60 seconds and try once more. If it still misses, report it as lag or failure, not success.

**Claude Code plugins.** Read the marketplace manifest for the plugin's entry. If the entry pins a `version` (or a `ref`/`sha` in its `source`), it must point at `<V>`/`<TAG>`. If the source is an unpinned git URL, the marketplace serves whatever the plugin repo's default branch holds, so the manifest has nothing to compare; surface 5's install is the readback. Say so under caveats.

## 5. Clean install smoke

Install the published artifact into a throwaway `HOME` and run it. Skip when the project publishes nothing installable (a tag-only release); list it as absent.

```bash
VERIFY_HOME=$(mktemp -d)
cd "$VERIFY_HOME"
```

| Ecosystem | Install | Check |
|---|---|---|
| npm CLI | `HOME="$VERIFY_HOME" npm exec --yes -- <pkg>@<V> --version` | prints `<V>` |
| PyPI | `python3 -m venv "$VERIFY_HOME/venv" && "$VERIFY_HOME/venv/bin/pip" install "<pkg>==<V>"` | `"$VERIFY_HOME/venv/bin/<cli>" --version` prints `<V>` |
| crates.io | `cargo install <crate> --version <V> --root "$VERIFY_HOME"` | `"$VERIFY_HOME/bin/<bin>" --version` prints `<V>` |
| Claude Code plugin | `HOME="$VERIFY_HOME" claude plugin marketplace add <marketplace-url>` then `HOME="$VERIFY_HOME" claude plugin install <plugin>@<marketplace>` | `HOME="$VERIFY_HOME" claude plugin list` shows `Version: <V>`, and `HOME="$VERIFY_HOME" claude plugin details <plugin>@<marketplace>` lists its components |

After `--version`, run one harmless command the package offers (`--help`, `--dry-run`, `plugin details`). Never run anything that writes outside `$VERIFY_HOME`.

A library with no CLI: import it instead (`node -e "require('<pkg>')"`, `python -c "import <mod>"`).

## Cleanup

```bash
rm -rf "$VERIFY_HOME" "$VERIFY_SRC"
```

Remove only the temp dirs this phase created, then `cd` back to the project root.

## Output

```
Is <TAG> released? yes | no

Evidence
- tag: origin <TAG> -> <sha> (matches local)
- release: <url>, published <publishedAt>, not draft, not prerelease, <n> assets
- versions at tag: <file>=<V>, ...
- registry: <what was read and what it said>
- install smoke: <what was installed, what --version printed>

Caveats
- absent by design: <surfaces the project doesn't ship>
- <diverged checkout and which live source replaced it, dist-tag or mirror lag, unpinned marketplace, Case C host>

Cleanup
- removed <temp dirs>
```

**yes** requires every shipped surface to pass. Any failure, a draft release, or a registry still missing `<V>` after the retry makes it **no**, with the failing surface named first.
