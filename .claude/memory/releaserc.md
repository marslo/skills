---
name: releaserc
description: "Self-contained semantic-release .releaserc.js changelog config: why Handlebars writerOpts (mainTemplate/headerPartial/commitPartial) are vendored, what each customizes, the (#N)-PR vs sha-link rule, h1 changelog-title reuse, and input→output samples. ~/.releaserc.js is the fallback config for repos with no rc of their own. Two invocation paths: GitHub workflow (needs repo-local rc) or CLI via ~/.marslo/bin/semantic-release. Reference before editing it."
metadata:
  node_type: memory
  type: reference
  modified: 2026-08-25T00:00:00.000Z
---

# semantic-release `.releaserc.js` — vendored changelog templates

Machine-wide reference for the hand-rolled changelog config. **`~/.releaserc.js` = the fallback** used when a repo has no rc of its own. Read this before touching it so the templates stay consistent.

## Invocation

Two ways a release runs; they resolve the rc differently:

| path | rc resolution |
|---|---|
| **GitHub Actions workflow** | semantic-release auto-discovers the **repo-local** `.releaserc*`. The repo **must** commit its own `.releaserc.js` — the workflow never sees `~/.releaserc.js`. |
| **CLI: `~/.marslo/bin/semantic-release`** | custom bash wrapper. rc lookup: cwd `.releaserc*` / `release.config.*` (auto-discovered) → repo root → **fallback `~/.releaserc.{js,json}` via `--extends`**. Dies if none found. |

The CLI wrapper (`~/.marslo/bin/semantic-release`) also:
- requires `GITHUB_TOKEN` | `GH_TOKEN` | `SEMANTIC_RELEASE_TOKEN` and `GITHUB_USER` (or git `user.name`)
- runs `npx semantic-release --branches <branch>` with `CI=true`, gpg signing off, `NODE_PATH=$(npm root -g)`
- flags: `-b/--branch <b>`, `-v/--verbose/--debug`, `--no-debug`, `--dryrun`, `-i/--init` (global-installs semantic-release + plugins + `conventional-changelog-conventionalcommits`), `-h/--help`

> [!NOTE]
> Fallback only supplies the **generic** config (CHANGELOG.md asset, `pre-commit run --files CHANGELOG.md || true`). A repo that needs version-file bumps or extra assets **must** ship its own `.releaserc.js` with repo-specific `prepareCmd`/`assets` — the fallback can't carry those.

## Why the templates are inlined (not stock)

`@semantic-release/release-notes-generator` renders via the **Handlebars** `conventional-changelog-writer`. `conventional-changelog-conventionalcommits >= 9` dropped its Handlebars `writerOpts` in favour of `@conventional-changelog/` templates that release-notes-generator does **not** use → with preset ≥ 9 the section grouping silently vanishes (flat list, no `### Features`).

> [!IMPORTANT]
> Verified empirically: running the **stock** preset (no custom `writerOpts`) under `release-notes-generator@14` + `conventionalcommits@10` throws `Missing helper: "conventional-changelog-conventionalcommits requires conventional-changelog-writer@9 or newer …"`. The bundled writer is too old to render the preset's new templates. **So vendoring `mainTemplate` / `headerPartial` / `commitPartial` is mandatory**, not a preference. The preset is then used only for its commit **parser**.

## Building blocks

| const | role |
|---|---|
| `SECTIONS` | ordered `[{type, section}]` — drives both `presetConfig.types` and the `### <section>` order |
| `TYPE_TO_SECTION` | `type → section label` map (used in `transform`) |
| `SECTION_ORDER` | section titles in intended order; unknown titles sort **last** |
| `isSignoff` / `stripSignoff` | drop `Signed-off-by:` trailers |
| `detectChangelogTitle` | reuse an existing `# ` h1 atop CHANGELOG.md (see [h1 title](#dynamic-changelog-title-h1-reuse)) |

## Customizations

### `mainTemplate`
Renders `{{> header}}`, then `### ⚠ <BREAKING>` note groups, then each commit group as `### {{title}}` + its commits. This is what restores the section grouping the stock preset loses.

### `headerPartial`
Version header line: `## [x.y.z](host/owner/repo/compare/prevTag...currTag) (date)` when `linkCompare`, else plain version. Uses `@root.host/owner/repository`.

### `commitPartial`  ← the most-edited piece
```
'*{{#if scopeFmt}} {{scopeFmt}}:{{/if}} {{#if subject}}{{subject}}{{else}}{{header}}{{/if}}' +
'{{#unless hasIssueRef}}{{#if @root.linkReferences}} ([{{shortHash}}]({{@root.host}}/{{@root.owner}}/{{@root.repository}}/commit/{{hash}})){{/if}}{{/unless}}' +
'\n{{#if body}}\n{{body}}\n{{/if}}\n{{#if footer}}\n\n{{footer}}\n{{/if}}\n'
```
- drops the `type:` keyword (it becomes the `### <section>` header via `transform`)
- renders `scopeFmt` (each comma-split scope bolded, built in `transform`) + `: `, keeps `{{subject}}` (falls back to `{{header}}` if no subject). `feat(a,b): x` → `**a**, **b**: x` (comma-split, trimmed, colon **outside** the bold — not `**a,b:**`).
- **PR vs push link rule** (via `hasIssueRef`): if the subject has an inline `(#N)`, GitHub auto-links it → leave the line link-free. If **no** `(#N)` (direct push), append `([shortHash](…/commit/<hash>))`.
- body/footer follow (built in `transform`)

> Do **not** re-add a `, closes {{#each references}}…` clause. It double-printed `#N` (inline `(#N)` **and** `, closes #N`) and produced a broken `host/owner//issues/N` URL (missing repo, because `this.repository` is empty for same-repo refs and the bundled writer doesn't backfill it).

### `transform(commit)` — the ordered steps
1. `c.type = TYPE_TO_SECTION[c.type]` → section label (drives grouping).
2. **restore `shortHash`**: `if (typeof c.hash === 'string') c.shortHash = c.hash.substring(0,7)`. The overridden transform loses the preset's default `shortHash`; without this the link text renders empty `[](…/commit/<hash>)`.
3. **`c.hasIssueRef = /\(#\d+\)/.test(c.subject || '')`** — only a *parenthesized* `(#N)` counts as a PR. `Closes #42` (no parens) does **not** trigger it → left in the body for GitHub to auto-link, and still gets a sha link.
4. **`c.scopeFmt`**: `if (c.scope) c.scopeFmt = c.scope.split(',').map(s => '**' + s.trim() + '**').join(', ')` — comma-split scopes, trim, bold each. The parser captures a comma scope as one string (`a,b`), so without this it renders `**a,b:**`; `scopeFmt` makes it `**a**, **b**`. Kept separate from `c.scope` so the note template's `{{commit.scope}}` isn't double-bolded.
5. fold `body` + `footer` (parser may split a trailing body bullet into `footer`), drop signoff; bullet body (`- …`) → 2-space sub-list, free-form body → blank line + 4-space verbatim block; `c.footer = null`.
6. drop signoff from `notes`.

### `writerOpts` sort
- `groupBy: 'type'`; `commitGroupsSort` ranks by `SECTION_ORDER` with unknown types → last; `commitsSort: ['header','subject']`.

### Dynamic changelog title (h1 reuse)
`detectChangelogTitle('CHANGELOG.md')` reads the first non-empty line; if it is a single `# ` ATX header, reuse it as `changelogTitle` so releases insert **below** it. Missing/none → leave unset (semantic-release just prepends). Wired via `["@semantic-release/changelog", Object.assign({changelogFile}, CHANGELOG_TITLE ? {changelogTitle} : {})]`.

## Samples (input → rendered)

| commit message | changelog line |
|---|---|
| `feat(keyword): description (#12)` | `* **keyword**: description (#12)` — verbatim, no link (GitHub links `(#12)`) |
| `feat(keyword): description` | `* **keyword**: description ([bbbbbbb](…/commit/…))` — sha appended |
| `feat(a, b , c): multi` | `* **a**, **b**, **c**: multi ([…])` — comma-split scopes, trimmed, each bolded |
| `chore: sync farm (#57)` | `* sync farm (#57)` — no scope, verbatim |
| `fix(api): correct thing` + `Closes #42` footer | `* **api**: correct thing ([ccccccc](…/commit/…))` + body line `Closes #42` (auto-linked) |
| `refactor(cli): add flag` + `- support --json` / `- validate input` | `* **cli**: add flag ([ddddddd](…))` + 2-space bullet sub-list |

Grouped under `### Features` / `### Bug Fixes` / `### Code Refactoring` / `### Others` in `SECTIONS` order.

## Verify a change (real pipeline)

Local `node` is old (v12, ESM-only preset fails). Use **node ≥ 18** + the global modules:

```bash
NODE_PATH=/opt/homebrew/lib/node_modules /opt/homebrew/bin/node verify.mjs
```
`verify.mjs` `import`s `@semantic-release/release-notes-generator`, loads the target `.releaserc.js`, and calls `generateNotes(rng, context)` with fake `commits` (each `{hash, message, committerDate}`) and `context.options.repositoryUrl`. This exercises the real writer incl. `finalizeContext` — the only reliable way to confirm the `(#N)` / sha / grouping behavior. Handlebars-only rendering hides the broken-URL / dedup edge cases.

Related: [[git-wildmatch-cheatsheet]] (unrelated, git glob).
