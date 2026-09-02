# Memory index

- [bartender-live-mode-mouse-warp](bartender-live-mode-mouse-warp.md) — Bartender 6 Live layout mode warps the mouse and breaks Chrome hover; fix is On-Demand mode
- [coc-pyright-completion-resolve-regression](coc-pyright-completion-resolve-regression.md) — since `6c1cf016`, pyright completion items lose documentation/detail (itemDefaults.data + applyKind not preserved); roll back to `2abe7008`
- [corp-ssh-github-rst-instability (en)](corp-ssh-github-rst-instability.en.md) — corp/office network kills GitHub SSH (port 22) via forged-RST injection; use HTTPS
- [corp-ssh-github-rst-instability (zh)](corp-ssh-github-rst-instability.zh.md) — 公司网络下 GitHub SSH(22) 被伪造 RST 掐断, 改用 HTTPS
- [devops-jenkins-development](devops-jenkins-development.md) — nvim+coc groovy/Jenkinsfile doc setup (ctags shared-libs, groovy-libs.sh, jenkins-libs.sh, gdoc offline javadoc)
- [git-diff-textconv](git-diff-textconv.md) — why/how stylus-*.json uses a git textconv diff driver for readable diffs
- [git-wildmatch-cheatsheet (en)](git-wildmatch-cheatsheet.en.md) — git config glob (includeIf/hasconfig/gitignore) = wildmatch, not POSIX fnmatch; token→regex table + tests
- [git-wildmatch-cheatsheet (zh)](git-wildmatch-cheatsheet.zh.md) — git config 的 glob = wildmatch 非 fnmatch; 含 token→正则表与实测矩阵
- [nvim-groovy-tags](nvim-groovy-tags.md) — vim/nvim+coc groovy/Jenkinsfile shared-lib usage: groovy_tags hover + arity-aware go-to-def + idle auto-hover
- [releaserc](releaserc.md) — semantic-release `.releaserc.js` changelog config (vendored Handlebars writerOpts, (#N)-PR rule); `~/.releaserc.js` fallback
- [vim-nvim-config-layout](vim-nvim-config-layout.md) — marslo's vim/nvim config file map (entry points, vimrc.d modules + load order, nvim lua/autoload, dotfiles repo); also the coc-html external-typescript fix
- [vnc-chrome-portal](vnc-chrome-portal.md) — marslo's TigerVNC/XFCE server: Chrome upload dialog fix, portal disabled at runtime, session gotchas
