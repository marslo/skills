---
name: nvim-groovy-tags
description: vim/nvim+coc editor USAGE for groovy/Jenkinsfile shared libs — groovy_tags cross-lib hover + arity-aware go-to-definition + idle auto-hover, repo .tags vs gdoc classpath, keys/config, nvim-only. environment setup is a separate memory.
metadata:
  type: reference
---

# vim/nvim + coc · groovy-lib editor tooling (usage)

The **editor-usage** side of the groovy/Jenkinsfile setup — how completion, hover, and
go-to-definition are driven in the buffer, plus keys/config and the nvim-only caveat.

> The **environment** side — building the classpath jars, the gdoc offline index, ctags,
> the download scripts, and the setup/troubleshooting runbook — is a **separate** memory:
> [[devops-jenkins-development]]. Kept apart on purpose (same origin, different concern:
> environment vs tool usage); this file references it, does not duplicate it.

Centers on the `groovy_tags` **module** built on top of the `[GT]` completion source, for
the release-engineering shared libs (`~/iMarslo/job/code/re/release-engineering/`:
`vars/*.groovy` + `jenkinsfile/`). groovyls cannot resolve the dynamically-injected globals
(`color`, `constant`, `wrapper`, `replica` …) or same-file overloads, so these are served
from `.tags` instead.

## Two tag indexes (kept independent)

| index | content | who reads it |
|---|---|---|
| **repo-root `.tags`** (via `gctags`, over `vars/*.groovy` + `jenkinsfile/`) | your shared libs | `[GT]` completion, `groovy_tags#hover/definition`, native `<C-]>` |
| **global/classpath** `~/.cache/nvim/gdoc/.tags` (+ `src/`, `javadoc-map.tsv`, ~88MB/438k) | jar symbols (groovy stdlib, jenkins core, `java.*`) | `,gd` (`gdoc#hover`), `<C-]>` into jar sources, coc generic `[Tag]` source |

`groovy_tags` **explicitly skips** the `~/.cache/nvim/gdoc/` tag file (it has no `vars/`),
so completion dropped 818ms→24ms per `.`. The classpath index is NOT wasted — it is
consumed by `,gd` / `<C-]>` / `[Tag]`, just not by the vars-lib feature.

## Three autoload files

| file | provides | source |
|---|---|---|
| `~/.config/nvim/autoload/coc/source/groovy_tags.vim` | `[GT]` `.`-completion + javadoc preview | repo `.tags` |
| `~/.config/nvim/autoload/groovy_tags.vim` | `K` hover, `gd` go-to-def, idle auto-hover, `<M-h>` toggle | repo `.tags` (+ live buffer for same-file calls) |
| `~/.config/nvim/autoload/gdoc.vim` | `,gd` classpath hover, `:GdocBuild` | `~/.cache/nvim/gdoc/` |

The two `groovy_tags.vim` share a base name by choice (same feature family), not by
requirement — separate namespaces `coc#source#groovy_tags#*` vs `groovy_tags#*`, neither
calls the other. `[GT]` (my libs) and coc `[Tag]` (all `&tags`) are two different sources.

## Features added (this session)

- `groovy_tags#hover()` (`K`) — javadoc float; falls back to coc/groovyls when the token isn't a resolvable lib call.
- `groovy_tags#definition()` (`gd`) — jump to the def; same-file jump skips `:edit` (avoids E37 on a modified buffer).
- **arity-aware overload selection** — counts args at the call site, picks the matching overload (exact, then varargs `argc >= count-1`); echoes the source line verbatim, so `String... files` stays exact (groovyls mis-renders it as `String; files`).
- resolves `lib.member` (cross-lib) AND bare `name( … )` (current buffer, else `vars/<name>.groovy` `call()`).
- **idle auto-hover** — `CursorHold` (→ `&updatetime`) by default; a private `CursorMoved` debounce timer when `g:groovy_tags_auto_hover_delay` (ms) is set. Only fires on resolvable lib calls (silent otherwise).
- `groovy_tags#toggle_auto_hover()` — flip on/off, echoes state.
- `groovy_tags#setup()` — per-buffer wiring, called from a `FileType groovy,Jenkinsfile` autocmd, idempotent across `:e`.

## Config (all live, no reload)

- `g:groovy_tags_auto_hover` — on/off; accepts `1/0`, `v:true/v:false`, `yes/no|true/false|on/off`.
- `g:groovy_tags_auto_hover_delay` — idle ms; unset (or -1) → follow `&updatetime`.
- keys: `K` hover, `gd` def, `<M-h>` / `:GroovyHoverToggle` toggle. (`,gd` = gdoc classpath hover; `<C-]>` = native tag jump, arity-unaware.)
- defaults set in `~/.marslo/vimrc.d/settings`; keys/command in `~/.marslo/vimrc.d/shortcuts`; wiring in `~/.marslo/vimrc.d/autocmd`.

## nvim-only (important)

`groovy_tags#*` and `gdoc#*` live under `~/.config/nvim/autoload`, which is on **nvim's**
runtimepath (via `init.lua`) but **NOT vim's** (`~/.vimrc` never adds `~/.config/nvim`).
So `groovy_tags#setup()` is wired in **`~/.config/nvim/init.lua`** (nvim-only file), not the
shared `~/.marslo/vimrc.d/autocmd` — plain vim never sees it (would raise **E117**). The
init.lua `FileType groovy,Jenkinsfile` autocmd guards with
`nvim_get_runtime_file('autoload/groovy_tags.vim', false)` before calling. Note:
`exists('*groovy_tags#setup')` can NOT be used as the guard — it returns 0 until the function
is already loaded (it does not autoload), so it would disable the feature.

Module audit: only nvim-specific piece is `s:float` (`nvim_open_win`/`nvim_create_buf`/
`nvim_buf_set_lines`/`nvim_win_close`), already guarded by `if !has('nvim') | echo …`.
Everything else (`matchstrpos`, `timer_start/stop`, `v:t_*`, autoload) is vim-compatible.
Moving to `~/.vim/autoload/` (loadable by both, since init.lua prepends `~/.vim`) is possible
but pointless: hover degrades to `:echo`, auto-hover would echo-spam on idle, and the `[GT]`
source + gdoc stay nvim-located — so the stack is nvim-only by design.

## Reference config files

`~/.config/nvim/coc-settings.json`, `~/.config/nvim/init.lua`, `~/.vimrc`,
`~/.config/nvim/autoload/coc/source/groovy_tags.vim`, `~/.ctags.d/groovy.ctags`.
README: `dotfiles/.marslo/vimrc.d/README.md` → "Groovy / Jenkinsfile documentation".
