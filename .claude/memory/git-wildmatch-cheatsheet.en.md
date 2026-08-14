---
name: git-wildmatch-cheatsheet
description: "git config glob (includeIf/hasconfig/gitignore) = wildmatch (WM_PATHNAME), not POSIX fnmatch; token→regex table, `**` rules, a verified pattern×URL matrix, runnable test harness, and GitHub-vs-Ruby fnmatch note. Project-agnostic."
metadata: 
  node_type: memory
  type: reference
  originSessionId: c5d533da-3e55-46c1-8a01-e745585d0e11
  modified: 2026-08-08T02:36:11.568Z
---

# git config glob = wildmatch (not fnmatch); `**` is the only thing that crosses `/`

Machine-wide reference. Chinese version: [[git-wildmatch-cheatsheet-zh]]. Applies to `gitignore`, pathspec, and `includeIf`/`hasconfig:remote.*.url` (used for per-account routing — see [[corp-ssh-github-rst-instability]]).

## Token → regex (anchored, `/`-aware / WM_PATHNAME)

| glob | regex | meaning |
|------|-------|---------|
| `*`   | `[^/]*`     | run of non-slash chars, within one segment |
| `?`   | `[^/]`      | **exactly one** non-slash char (NOT zero-or-one) |
| `*/`  | `[^/]*/`    | exactly **one** segment, then `/` |
| `**/` | `([^/]*/)*` | **zero or more** segments (any depth) |
| `/**` | `/.*`       | everything below (trailing) |
| `[…]` | char class  | brackets do **not** match `/` in path mode (`[:/]` matches `:` but not `/`) |

## Key rules

- Only a **literal `/`** and the `**` forms (`**/`, `/**`, `/**/`) cross `/`. `*`, `?`, `[…]` **never** match `/`.
- `**` crosses only as a **standalone path component** (`**/`). Glued to text (`**foo`, `foo**`) it degrades to a plain `*` (does not cross `/`).
- `?` = exactly one non-`/` char. Handy for the `:` in scp URLs: `*github*.com?marslo/**` matches `git@github.com:marslo/…`.
- `hasconfig:remote.*.url:` matches the **raw stored** `remote.origin.url` (insteadOf/pushInsteadOf do NOT change it).

## Verified pattern × URL-form matrix (account "marslo", git 2.55)

| pattern | `https://…/marslo/` | `git@github.com:marslo/` | `git@github-marslo.com:marslo/` | `ssh://git@github.com/marslo/` | `gitlab` |
|---|:--:|:--:|:--:|:--:|:--:|
| `**/marslo/**`            | ✓ | · | · | ✓ | ✓ ⚠ host-agnostic (matches any host) |
| `**:marslo/**`            | · | ✓ | ✓ | · | ✓ ⚠ host-agnostic |
| `**/*github*.com/marslo/**` | ✓ | · | · | ✓ | · host-scoped |
| `*github*.com?marslo/**`  | · | ✓ | ✓ | · | · host-scoped |

Notes:
- The split axis is the **separator before the org**: `/` (https + `ssh://…/`) vs `:` (scp `git@host:org`). No single glob covers both → use two.
- `**/github.com/…` fails for `ssh://` because the host segment is `git@github.com` (not `github.com`); use `**/*github*.com/…` so the leading `*` absorbs the `git@`.
- Host-scoped (contains `github`) avoids leaking your identity to a same-named org on another host (e.g. gitlab); host-agnostic (`**/marslo/**`) is simpler but leaks.

## Runnable test harness

```bash
GIT=git   # or /opt/homebrew/bin/git
match() {           # match '<glob>' '<url>' -> ✓ / ·
  local pat="$1" url="$2" H TMP r
  H="$(mktemp -d)"; TMP="$(mktemp -d)"
  printf '[user]\n  email=NO\n[includeIf "hasconfig:remote.*.url:%s"]\n  path=%s/h\n' "$pat" "$H" > "$H/gc"
  printf '[user]\n  email=YES\n' > "$H/h"
  ( cd "$TMP"; "$GIT" init -q; "$GIT" remote add origin "$url"
    r="$(GIT_CONFIG_GLOBAL=$H/gc GIT_CONFIG_SYSTEM=/dev/null "$GIT" config user.email)"
    test YES = "$r" && echo '✓' || echo '·' )
  rm -rf "$H" "$TMP"
}

# build a matrix
pats=( '**/marslo/**' '**:marslo/**' '**/*github*.com/marslo/**' '*github*.com?marslo/**' )
urls=( 'https://github.com/marslo/x.git' 'git@github.com:marslo/x.git'
       'git@github-marslo.com:marslo/x.git' 'ssh://git@github.com/marslo/x.git'
       'git@gitlab.com:marslo/x.git' )
for p in "${pats[@]}"; do printf '%-28s' "$p"
  for u in "${urls[@]}"; do printf ' %s' "$(match "$p" "$u")"; done; echo
done
```

## GitHub vs Ruby vs POSIX fnmatch

- **git** uses its own `wildmatch` (`wildmatch.c`), documented in `gitignore(5)` "PATTERN FORMAT" and `git-config(5)` (`includeIf`/`hasconfig`). It is fnmatch-family **plus** the `**/`, `/**` extensions.
- **GitHub** repository rulesets say "fnmatch syntax" — that is **Ruby `File.fnmatch`** (GitHub is Rails). With `File::FNM_PATHNAME` it reproduces the same `/`-boundary behavior above and also supports `**/`. (Docs: GitHub "Using fnmatch syntax".)
- **POSIX `fnmatch(3)` + `FNM_PATHNAME`** is the common base but has **no `**`**.
- Net: git-wildmatch ≈ Ruby-`File.fnmatch(…, FNM_PATHNAME)` ≈ GitHub-rulesets-fnmatch; plain POSIX fnmatch is the subset without `**`.
