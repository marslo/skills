---
name: git-message
description: >-
  Generate a git commit message that follows the Conventional Commits /
  semantic-release format, providing both a single-line and a multi-line
  version. Works on the working tree or on an existing commit/tag/ref. Use
  when the user runs /git-message, /git-message <ref>, or asks for a commit
  message for staged, unstaged, or already-committed changes.
disable-model-invocation: true
---

# git-message

Produce a commit message in **Conventional Commits / semantic-release** format.
Always output **two** forms: a single-line summary and a full multi-line message. Only write the message — do **not** run `git commit` unless the user explicitly asks.

**CRITICAL RULE**:

If not explicitly asked to commit, !!NEVER!! run git commit for any message. Provide the message if necessary, but unless explicitly instructed, you must absolutely not run git commit!
如果没有明确的让你提交, !!切勿!! git commit 任何 message. 如果有必要提供就行, 除非我显式的让你提交, 否则一律不得 git commit!

## Input

- `/git-message` — summarize the **working tree** (staged, else unstaged).
- `/git-message <ref>` — summarize the changes introduced by an existing commit/tag/reference, e.g. `HEAD^^`, a short hash, or a tag name.
- `/git-message <path>` — summarize the changes to a specific file or directory in the working tree.
- `/git-message <path> <ref>` — summarize the changes to a specific file or directory introduced by an existing commit/tag/reference.

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
2. Pick the `type`, optional `scope`, and identify the **dominant action** of the
   change as a whole — the single verb that best names what happened overall
   (e.g. `merge`, `combine`, `unify`, `move`, `rename`, `split`, `drop`, `rewrite`).
3. Write the **multi-line** version first (subject + concise bullets). Then distill
   the **single-line** as a one-line, high-level summary of that **whole** message —
   built around the dominant action, **not** a copy of the multi-line's subject/first
   line. The single-line has **no** width limit (see "Single-line vs multi-line" below).

## Format

```
<type>(<scope1,scope2,...>): <subject>

- <key change 1>
- <key change 2>
- ...

<footer>
```

Rules:

- **Subject**: imperative mood, lowercase, no trailing period. The ≤ 50 char
  (hard ≤ 72) length cap applies to the **multi-line subject**; the **single-line
  is exempt** from any width limit (see "Single-line vs multi-line").
- **Scope** (optional): the area(s) touched. When the diff spans multiple areas,
  use **comma-separated scopes** — `refactor(cli,libs): …`. Omit the scope rather
  than invent a vague one.
- **Body** (multi-line only): a **short bulleted list** of the key changes — one
  `- ` markdown item each, terse and imperative. Each bullet **starts directly with
  a lowercase verb phrase** (`update docs`, `fix bug`), no trailing period. Do
  **not** prefix a bullet with a single-letter label or initial. Capture *what
  changed at a high level*, **not** a file-by-file rundown: do **not** list
  filenames, include code, or over-explain. Aim for ~2–5 bullets; drop the body
  entirely if the subject already says it all.
- **Ignore incidental noise**: do not create a subject or bullet for churn that
  isn't a real change — header/metadata bumps (`LastChange:`/`Updated:` timestamps),
  auto-formatting reflow, or generated-file diffs. Summarize the actual change only.
- **Footer** (when relevant): `BREAKING CHANGE: <desc>`, and issue refs like `Closes #123`.
  Only add an issue ref when the user provides a number (e.g. `/git-message closes #142`)
  or it's clearly in the branch name (e.g. `feature/142-...`, `PROJ-142-...`).
  **Never invent an issue number** — omit the footer if none is known.
- A breaking change is marked either with `!` after the type/scope (`feat(api)!:`) or a `BREAKING CHANGE:` footer.

### Single-line vs multi-line

The two forms are **not** the same string. The single-line is a **high-level
summary of the *entire* multi-line message** (its subject **and** its bullets),
condensed into one line — **not** a variant of the multi-line's first line/subject.

- **Single-line** — distill the whole multi-line into one line (the overall "what
  happened"), led by the dominant action verb from step 2. **No width limit**: it
  does *not* obey the ≤ 50/≤ 72 subject cap and is never wrapped — make it as long
  as it needs to be to capture the essence, but keep it a single line and a valid
  Conventional Commit. Prefer high-level "what happened" over enumerating specifics.
- **Multi-line subject** — its own subject line; obeys the ≤ 50/≤ 72 length cap and
  imperative mood. Do **not** reuse it verbatim as the single-line.
- Both must be valid Conventional Commits. If the single-line is just the multi-line
  subject copied, re-derive it as a summary of the whole message instead.

<good-example>
Single-line: `refactor(install): merge cli + skill installers into one script`
Multi-line subject: `refactor(install)!: unify installers into root install.sh with modes`
</good-example>

<bad-example>
Single-line is just the multi-line's first line copied verbatim (no distillation).
</bad-example>

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

### `docs` vs `feat`/`fix`/`refactor`: classify by the file's role, not its extension

Do **not** reason "it's markdown / prose, therefore `docs`". `docs` is only for
**human-facing, ancillary documentation** — `README`, guides, `docs/…`, usage /
install notes, code comments, CHANGELOG prose: material a program never executes.

If the changed file is itself the **product or the executable / behavioral source**
— e.g. a `SKILL.md` an agent runs, a Cursor/agent rule, a prompt template, or a
config / manifest that drives behavior — classify it by its **effect**, exactly as
you would for code, even though it is markdown:

- `feat` — adds a new capability / instruction.
- `fix` — corrects wrong or broken behavior.
- `refactor` — restructures with **no** behavior change (e.g. splitting a big file,
  moving sections into includes, renaming for clarity).

<good-example>
Splitting a large `SKILL.md` into a lean core + `references/` with no behavior
change → `refactor(skill): split SKILL.md into a lean core + references/`.
Editing `README.md` to describe that same skill → `docs: …`.
</good-example>

<bad-example>
Classifying a `SKILL.md`/prompt/rule restructure as `docs` just because the file
has a `.md` extension.
</bad-example>

## Output template

Respond with exactly these two blocks. The single-line is a distilled summary,
**not** a copy of the multi-line subject (they may differ):

**Single-line**
```
<type>(<scope>): <subject>
```

**Multi-line**
```
<type>(<scope>): <subject>

- <key change>
- <key change>

<optional footer: BREAKING CHANGE / Closes #NN>
```

## Example

Input: added a JWT login endpoint and token-validation middleware.

Note how the single-line distills to the essence (`add JWT auth`) while the
multi-line subject carries the specifics — they are intentionally different.

**Single-line**
```
feat(auth): add JWT authentication
```

**Multi-line**
```
feat(auth): add JWT login endpoint and token-validation middleware

- issue signed JWTs from a new /login route
- validate tokens on protected routes via middleware
- replace the ad-hoc session check

Closes #142
```
