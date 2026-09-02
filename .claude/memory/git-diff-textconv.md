---
name: git-diff-textconv
description: why/how stylus-*.json uses a git textconv diff driver to get readable diffs
metadata: 
  node_type: memory
  type: reference
  originSessionId: f91ca9f6-c568-489c-913f-52b35898824c
  modified: 2026-08-26T07:07:14.038Z
---

Stylus browser-extension exports (`stylus-*.json`, e.g. `~/Desktop/stylus-2026-08-25-23-11.json`) are minified: the whole document sits on one very long line, and CSS bodies are stored as string values with `\n` escaped inline. A raw `git diff` shows the entire file as a single changed line and is unreadable.

Fix — a git **textconv** diff driver (diff-only, does NOT change what is committed; the stored blob stays minified):

- `~/.gitattributes`: `stylus-*.json  diff=stylus-json` — routes matching paths to the driver.
- gitconfig: `[diff "stylus-json"] textconv = ~/.marslo/gitconfig.d/bin/stylus-expand.sh` and `cachetextconv = true`.
- `stylus-expand.sh`: `jq '.' "$1" | sed 's/\\n/\n/g' || cat "$1"` — jq pretty-prints the JSON structure, sed turns the escaped `\n` inside CSS strings into real newlines, cat is the fallback if jq fails.

Principle: for diff purposes git pipes each version's blob through `textconv` and diffs the *converted* text instead of the raw bytes, so a one-liner expands into a line-by-line comparable form. `cachetextconv = true` caches the converted output keyed by blob SHA, so repeat diffs skip re-running the script.
