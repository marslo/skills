---
name: git-wildmatch-cheatsheet-zh
description: "git config 的 glob(includeIf/hasconfig/gitignore)= wildmatch(WM_PATHNAME),非 POSIX fnmatch;含 token→正则表、`**` 规则、实测的 pattern×URL 矩阵、可运行测试脚本、以及 GitHub 与 Ruby fnmatch 的区别。与项目无关。"
metadata: 
  node_type: memory
  type: reference
  originSessionId: c5d533da-3e55-46c1-8a01-e745585d0e11
  modified: 2026-08-08T02:36:40.624Z
---

# git config 的 glob = wildmatch(非 fnmatch);只有 `**` 能跨 `/`

机器级参考。英文版:[[git-wildmatch-cheatsheet]]。适用于 `gitignore`、pathspec、以及 `includeIf`/`hasconfig:remote.*.url`(多账号身份路由用它 —— 见 [[corp-ssh-github-rst-instability-zh]])。

## Token → 正则(锚定、按 `/` 分段 / WM_PATHNAME)

| glob | 正则 | 含义 |
|------|------|------|
| `*`   | `[^/]*`     | 段内任意多个非斜杠字符 |
| `?`   | `[^/]`      | **恰好一个**非斜杠字符(不是"零或一") |
| `*/`  | `[^/]*/`    | 恰好**一段**,再一个 `/` |
| `**/` | `([^/]*/)*` | **零或多段**(任意深度) |
| `/**` | `/.*`       | 其下所有(结尾用) |
| `[…]` | 字符类      | path 模式下**不**匹配 `/`(`[:/]` 匹配 `:` 但不匹配 `/`) |

## 关键规则

- 只有**字面 `/`** 和 `**` 形式(`**/`、`/**`、`/**/`)能跨 `/`。`*`、`?`、`[…]` **永不**匹配 `/`。
- `**` 只有作为**独立路径段**(`**/`)才跨 `/`;一旦和文字粘着(`**foo`、`foo**`)就退化成普通 `*`(不跨 `/`)。
- `?` = 恰好一个非 `/` 字符;正好用来顶 scp URL 里的 `:`:`*github*.com?marslo/**` 命中 `git@github.com:marslo/…`。
- `hasconfig:remote.*.url:` 匹配的是**存储的 raw** `remote.origin.url`(insteadOf/pushInsteadOf **不**改动它)。

## 实测 pattern × URL 形式 矩阵(账号 "marslo",git 2.55)

| pattern | `https://…/marslo/` | `git@github.com:marslo/` | `git@github-marslo.com:marslo/` | `ssh://git@github.com/marslo/` | `gitlab` |
|---|:--:|:--:|:--:|:--:|:--:|
| `**/marslo/**`            | ✓ | · | · | ✓ | ✓ ⚠ 不锁 host(任意 host 都中) |
| `**:marslo/**`            | · | ✓ | ✓ | · | ✓ ⚠ 不锁 host |
| `**/*github*.com/marslo/**` | ✓ | · | · | ✓ | · 锁 github |
| `*github*.com?marslo/**`  | · | ✓ | ✓ | · | · 锁 github |

要点:
- 分类轴是 **org 前面的分隔符**:`/`(https + `ssh://…/`)vs `:`(scp `git@host:org`)。单条 glob 覆盖不了两者 → 用两条。
- `**/github.com/…` 对 `ssh://` 不命中,因为 host 段是 `git@github.com`(不是 `github.com`);用 `**/*github*.com/…`,开头的 `*` 吸掉 `git@`。
- 锁 host(含 `github`)可避免把身份泄漏到别的 host 上同名 org(如 gitlab);不锁 host(`**/marslo/**`)更简单但会泄漏。

## 可运行测试脚本

```bash
GIT=git   # 或 /opt/homebrew/bin/git
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

# 生成矩阵
pats=( '**/marslo/**' '**:marslo/**' '**/*github*.com/marslo/**' '*github*.com?marslo/**' )
urls=( 'https://github.com/marslo/x.git' 'git@github.com:marslo/x.git'
       'git@github-marslo.com:marslo/x.git' 'ssh://git@github.com/marslo/x.git'
       'git@gitlab.com:marslo/x.git' )
for p in "${pats[@]}"; do printf '%-28s' "$p"
  for u in "${urls[@]}"; do printf ' %s' "$(match "$p" "$u")"; done; echo
done
```

## GitHub vs Ruby vs POSIX fnmatch

- **git**:用自研 `wildmatch`(`wildmatch.c`),文档在 `gitignore(5)` "PATTERN FORMAT" 和 `git-config(5)`(`includeIf`/`hasconfig`)。属 fnmatch 家族 **+** `**/`、`/**` 扩展。
- **GitHub** 仓库 rulesets 说的 "fnmatch syntax" = **Ruby `File.fnmatch`**(GitHub 是 Rails)。加 `File::FNM_PATHNAME` 时,`/` 边界行为与上表一致,也支持 `**/`。(文档:GitHub "Using fnmatch syntax"。)
- **POSIX `fnmatch(3)` + `FNM_PATHNAME`** 是共同基础,但**没有 `**`**。
- 结论:git-wildmatch ≈ Ruby-`File.fnmatch(…, FNM_PATHNAME)` ≈ GitHub-rulesets-fnmatch;纯 POSIX fnmatch 是去掉 `**` 的子集。
