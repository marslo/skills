---
name: vim-nvim-config-layout
description: marslo's vim/nvim config file map — entry points (~/.vimrc, init.lua), the ~/.marslo/vimrc.d/ modules and load order, nvim lua/config + autoload, coc/ctags files, dotfiles repo. use to resolve config paths without asking. also the coc-html external-typescript dependency fix (keep global TS 7 while coc-html uses 6).
metadata:
  type: reference
---

# vim / nvim config layout (marslo)

Where marslo's editor config lives, so paths don't need re-stating. Shared core (vimrc.d)
drives **both** vim and nvim; nvim layers Lua on top.

## Entry points

| file | editor | role |
|---|---|---|
| `~/.vimrc` | vim + nvim | main entry; sources `~/.marslo/vimrc.d/*` in order (below) |
| `~/.config/nvim/init.lua` | nvim only | prepends `~/.vim` to rtp, `source ~/.vimrc`, then loads `lua/config/*`; also nvim-only autocmds (e.g. `groovy_tags#setup`) |
| `~/.config/nvim/coc-settings.json` | nvim (coc) | coc + LSP settings (groovyls classpath, linters, sources, diagnostics) |

`~/.vimrc` load order: `os` (defines `IsNvim`/`IsVim`/`IsMac`/`IsWSL`/`IsWindows`) → plugins →
`extension` → `extra-extension` (**vim-only**, `has('vim')`) → `functions` → `cmds` → `theme` →
`settings` → `shortcuts` → `autocmd` → `highlight` → `unix` (non-WSL/non-Mac).

## `~/.marslo/vimrc.d/` (the modular core — vim + nvim)

| file | holds |
|---|---|
| `os` | OS/editor detection helpers (`IsNvim`, `IsVim`, `IsMac`, `IsWSL`, `IsWindows`) |
| `settings` | general options (indent, fold, encoding, `tags`, `updatetime`, `g:groovy_tags_*` …) |
| `shortcuts` | key maps + command abbreviations (`K`, `gd`, `<M-h>`, `<leader>gd`, `:GdocBuild`, `:GroovyHoverToggle` …) |
| `autocmd` | filetype autocmds (groovy/Jenkinsfile, sh, python, markdown, ale, groovy-LS-by-size …) |
| `extension` | plugin configs + keymaps (fzf, coc, ale, gitgutter, hexokinase, lexima …) |
| `extra-extension` | **vim-only** extras (tagbar, table-mode …) |
| `functions` | utility functions (`TabMessage`, `GetPlug`, lint helpers …) |
| `cmds` | custom commands + command abbreviations |
| `theme` | colorscheme, airline, UI |
| `highlight` | custom highlight groups / overrides |
| `devicons` | vim-devicons icon map per filetype (vim side) |
| `unix` | unix redraw/clipboard |
| `snips/` | coc-snippets `*.snippets` (e.g. `Jenkinsfile.snippets`) |
| `README.md` | documents the above (shortcut tables, plugin list) |

## nvim-only

- `~/.config/nvim/lua/config/*.lua` — plugin configs loaded by `init.lua`: `devicons`, `fzf`,
  `nvim-treesitter`(+`-textobjects`), `oil`, `cmp`, `copilot`, `snippets`, `lsp`, `theme`, `rainbow`.
- `~/.config/nvim/autoload/` — `coc/source/groovy_tags.vim` (`[GT]` completion), `groovy_tags.vim`
  (hover/go-to-def/auto-hover), `gdoc.vim` (classpath hover). all nvim-rtp only. see [[nvim-groovy-tags]].

## vim-only / plugins

- `~/.vim/plugged/` — vim-plug plugin install dir (`set runtimepath+=~/.vim/plugged`).
- `~/.vim/autoload/plug.vim` — vim-plug. `~/.vim/autoload/` is on **both** vim and nvim rtp
  (nvim via `init.lua` prepend), so it is the shared-autoload location.

## ancillary

- `~/.ctags.d/groovy.ctags` — universal-ctags groovy langdef (used by `gctags` + gdoc).
- `~/.groovylintrc.json` — npm-groovy-lint config (referenced by coc diagnostic-languageserver).
- caches: `~/.cache/nvim/gdoc/` (gdoc index), repo-root `.tags` (shared libs, via `gctags`).

## versioning

Tracked in the dotfiles git repo at `~/iMarslo/tools/git/marslo/dotfiles` (its work-tree is
`$HOME`, so repo paths `.marslo/…` and `.config/nvim/…` map directly to `~/.marslo/…` and
`~/.config/nvim/…`). Editing `~/.marslo/vimrc.d/settings` shows up as a change there.

## coc-html — external typescript dependency (verified 2026-09-01)

- `coc-html` 1.9.0 declares a runtime dep `typescript: ^6.0.3` but coc does not install it → server crashes with `Cannot find module 'typescript'`, then (with a wrong TS major) `Cannot read properties of undefined (reading 'JS')` at `ScriptKind.JS`.
- every coc-html version needs external typescript (1.7.0/1.8.0 → `^4.3`, 1.9.0 → `^6.0.3`); npm has only 6.0.2 and 6.0.3 in the 6.x line, so 1.9.0 resolves to 6.0.3.
- TS 7.x (e.g. 7.0.2, the native rewrite) is API-incompatible with coc-html 1.9.0.
- adding `typescript` to `g:coc_global_extensions` does NOT work — coc rejects any package lacking an `engines.coc` field.
- coc-tsserver bundles its own typescript (5.9.3) in its nested `node_modules`, invisible to coc-html.

### fix (two options)

1. local (preferred — keeps global TS 7 for other tools):
   ```
   cd ~/.config/coc/extensions && npm install typescript@6
   npm install -g typescript@7
   ```
   coc-html resolves `extensions/node_modules/typescript` (6.x) — earlier in the require walk than any `NODE_PATH` fallback; global `tsc` stays 7.x via PATH.

2. global (simplest — global tsc becomes 6.x too):
   ```
   npm install -g typescript@6
   ```
   needs `export NODE_PATH="/opt/homebrew/lib/node_modules"` (= `npm root -g`) so coc-html's `require('typescript')` reaches the global dir; `require` never searches the global dir on its own.

**Why:** `require('typescript')` walks `node_modules` up from the requiring file, then `NODE_PATH`; the global npm dir is not on that path. PATH only helps CLI binaries, not library `require`.

**How to apply:** on `Cannot find module 'typescript'` / `reading 'JS'` from coc-html, install a TS version matching coc-html's declared range (currently 6.0.3) into a spot the require walk reaches. Most robust variant: a dedicated dir `~/.config/coc/ts6` with `typescript@6.0.3` and `NODE_PATH` pointing there — coc never prunes it, unlike `extensions/node_modules`. After installing, `:CocRestart`.

Related: [[nvim-groovy-tags]] (groovy-lib editor tooling), [[devops-jenkins-development]] (env setup).
