---
name: git-commit
description: >-
  Generate a git commit message that follows the Conventional Commits /
  semantic-release format, providing both a single-line and a multi-line
  version. Works on the working tree or on an existing commit/tag/ref. Use
  when the user runs /git-commit, /git-commit <ref>, or asks for a commit
  message for staged, unstaged, or already-committed changes.
disable-model-invocation: true
---

# git-commit

Produce a commit message in **Conventional Commits / semantic-release** format.
Always output **two** forms: a single-line summary and a full multi-line
message. Only write the message — do **not** run `git commit` unless the user
explicitly asks.

## Input

- `/git-commit` — summarize the **working tree** (staged, else unstaged).
- `/git-commit <ref>` — summarize the changes introduced by an existing
  commit/tag/reference, e.g. `HEAD^^`, a short hash, or a tag name.

## Steps

1. Gather the diff to summarize.

   **With a `<ref>` argument** (already-committed changes):
   ```bash
   git show --stat <ref>
   git show <ref>                    # or: git diff <ref>^..<ref>
   git log -1 --format=%B <ref>      # the existing message, for reference
   ```
   Reuse any issue ref already present in the existing message. If `<ref>`
   resolves to a merge commit or a range, say so and summarize accordingly.

   **Without an argument** (working tree):
   ```bash
   git diff --staged --stat
   git diff --staged
   ```
   If nothing is staged, fall back to `git diff` / `git status` and mention that
   nothing is staged yet.
2. Pick the `type`, optional `scope`, and an imperative subject from the diff.
3. Emit the single-line version, then the multi-line version (templates below).

## Format

```
<type>(<optional scope>): <subject>

<body>

<footer>
```

Rules:

- **Subject**: imperative mood, lowercase, no trailing period, ≤ 50 chars (hard ≤ 72).
- **Body** (multi-line only): wrap at ~72 cols; explain the *why*, not the *what*.
- **Footer** (when relevant): `BREAKING CHANGE: <desc>`, and issue refs like `Closes #123`.
  Only add an issue ref when the user provides a number (e.g. `/git-commit closes #142`)
  or it's clearly in the branch name (e.g. `feature/142-...`, `PROJ-142-...`).
  **Never invent an issue number** — omit the footer if none is known.
- A breaking change is marked either with `!` after the type/scope (`feat(api)!:`) or a `BREAKING CHANGE:` footer.

### Types and semantic-release impact

Release impact reflects `~/.releaserc.js` (angular preset + custom `releaseRules`):

| type       | use for                                   | release |
|------------|-------------------------------------------|---------|
| `feat`     | a new feature                             | minor   |
| `fix`      | a bug fix                                 | patch   |
| `perf`     | a performance improvement                 | patch   |
| `refactor` | code change that isn't a feature or fix   | patch   |
| `docs`     | documentation only                        | patch   |
| `ci`       | CI configuration                          | patch   |
| `chore`    | maintenance, tooling, misc                | patch   |
| `revert`   | revert a previous commit                  | patch   |
| `style`    | formatting, whitespace, no logic change   | none    |
| `test`     | adding or fixing tests                    | none    |
| `build`    | build system or dependencies              | none    |

A `BREAKING CHANGE:` footer (or `!` after the type/scope) forces a **major** regardless of type.
Only `feat fix refactor chore docs perf ci` appear as CHANGELOG sections; `style test build revert` don't (though `revert` still triggers a patch).

> The `feat`→minor and breaking→major rows assume the two overrides in `~/.releaserc.js` (`{ type: 'feat', release: 'patch' }` and `{ breaking: true, release: 'minor' }`) stay **commented out**. If they're enabled, `feat` becomes patch and breaking becomes minor — update this table to match.

## Output template

Respond with exactly these two blocks:

**Single-line**
```
<type>(<scope>): <subject>
```

**Multi-line**
```
<type>(<scope>): <subject>

<body explaining why>

<optional footer: BREAKING CHANGE / Closes #NN>
```

## Example

Input: added a JWT login endpoint and token-validation middleware.

**Single-line**
```
feat(auth): add JWT login endpoint and token validation
```

**Multi-line**
```
feat(auth): add JWT login endpoint and token validation

Introduce a /login route that issues signed JWTs and middleware that
validates them on protected routes, replacing the ad-hoc session check.

Closes #142
```
