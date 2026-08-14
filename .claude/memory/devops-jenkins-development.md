---
name: devops-jenkins-development
description: nvim+coc groovy/Jenkinsfile doc setup — ctags shared-libs, groovy-libs.sh, jenkins-libs.sh, gdoc offline javadoc index, coc/lexima config
metadata:
  type: reference
---

# DevOps · Jenkins/Groovy Development Environment (nvim + coc)

Goal: in nvim, get **documentation (javadoc)** for groovy stdlib, jenkins-core, jenkins
plugin jars, and the release-engineering shared library (`vars/*.groovy`), in both
`groovy` and `Jenkinsfile` buffers.

Shared-libs repo: `~/iMarslo/job/code/re/release-engineering/` (`vars/` = groovy filetype,
`jenkinsfile/` = Jenkinsfile filetype).

## Three doc layers (which tool serves which scenario)

| scenario | provider | how |
|---|---|---|
| `vars/*.groovy` (real groovy) | **coc-groovy LSP** (referencedLibraries classpath) | completion + `K` hover |
| `Jenkinsfile` → `gh.cliRun` etc. | **groovy_tags** coc source (`[GT]`, `.tags`) | `.` completion **with javadoc** in the `info` float |
| any classpath symbol (groovy/Jenkinsfile) | **gdoc** offline index | cursor on symbol → `<leader>gd` → javadoc float |

> [!IMPORTANT]
> Jenkinsfile is **NOT** mapped to the groovy LSP. `@Library('..') _`, `pipeline{}` etc.
> are not valid standalone groovy → the LSP would raise false "unable to resolve class
> Library" diagnostics. So `g:coc_filetype_map` stays `{ 'yaml.ansible': 'ansible' }`
> (jenkinsfile absent). Jenkinsfile docs come from `groovy_tags` + `gdoc`, not the LSP.

## Files (created / modified)

| file | role |
|---|---|
| `~/.config/nvim/coc-settings.json` | `groovy.project.referencedLibraries` (LSP classpath), `suggest.insertMode: insert` |
| `~/.config/nvim/autoload/coc/source/groovy_tags.vim` | coc source: `gh.method` completion in Jenkinsfile, `info` = javadoc from `vars/*.groovy` |
| `~/.config/nvim/autoload/gdoc.vim` | `gdoc#hover()` (K-doc for classpath symbols), `gdoc#build()` |
| `~/.marslo/bin/lsp-gdoc` | offline javadoc indexer (extract sources jars + ctags + javadoc-map) |
| `~/.marslo/vimrc.d/shortcuts` | `<leader>gd` → `gdoc#hover()`, `:GdocBuild` |
| `~/.marslo/vimrc.d/extension` | `g:coc_filetype_map`, lexima suppress regex, coc `<CR>`/`<C-j>` maps |
| `~/.ctags.d/groovy.ctags` | universal-ctags groovy langdef (used by `gctags` and by gdoc's ctags run) |
| `~/.marslo/bin/ifunc.sh` | `gctags` function — generates the repo `.tags` for the shared libs |
| `/opt/groovy/groovy-libs.sh` | download groovy core + system-lib sources/javadoc + extensions |
| `/opt/jenkins/jenkins-libs.sh` | download jenkins war + plugins + core + extensions + bundled-jar docs |

## Data flow

```
groovy-libs.sh  ─► /opt/groovy/latest/*            ─┐
jenkins-libs.sh ─► ~/.groovy/lib/* (--ln)           ├─► referencedLibraries (coc-groovy LSP classpath)
                   /opt/jenkins/latest/WEB-INF/lib ─┘        │
                                                             └─► lsp-gdoc --build ─► ~/.cache/nvim/gdoc ─► <leader>gd
gctags (ifunc.sh) ─► release-engineering/.tags ─► groovy_tags coc source (Jenkinsfile gh.method + javadoc)
```

## Setup from scratch (also: after `--cleanup`)

```bash
# 1. groovy stdlib + all system-lib third-party jars' sources/javadoc + extensions
bash /opt/groovy/groovy-libs.sh --groovy --with-libs

# 2. jenkins war + plugins + core + extensions; fetch bundled-jar docs; link into ~/.groovy/lib
bash /opt/jenkins/jenkins-libs.sh --lts --sources --javadoc --ln

# 3. build the offline javadoc index (scans the sources jars downloaded above)
bash ~/.marslo/bin/lsp-gdoc --build
```

Then in nvim: `:CocRestart`. Order matters — 1 & 2 stage the jars, 3 indexes them.

Maintenance: after re-running a download script (new version/plugin), re-run
`lsp-gdoc --build` (or `:GdocBuild` in nvim). Shared-lib changes → regenerate `.tags` via `gctags`.
Full rebuild / free the cache: `lsp-gdoc --clean && lsp-gdoc --build` — `clean` wipes
`~/.cache/nvim/gdoc` (src + .tags + javadoc-map; reports freed size), the downloaded jars stay.

Dependencies (all present): `curl unzip jq ctags pandoc`, **bash >= 4.3**.

## Health checks (verify each layer)

```bash
# referencedLibraries dirs exist & populated
ls /opt/groovy/latest ~/.groovy/lib /opt/jenkins/latest/WEB-INF/lib | head
# gdoc index built (should print a big number)
grep -vc '^!' ~/.cache/nvim/gdoc/.tags
# shared-lib tags present (for Jenkinsfile gh.method)
ls -l ~/iMarslo/job/code/re/release-engineering/.tags
# a specific doc resolves end-to-end (sources jar + javadoc extraction)
ls ~/.cache/nvim/gdoc/src/org/codehaus/groovy/runtime/DefaultGroovyMethods.java
```

In nvim: `:CocList services` → `groovy` server **running** (attaches to `groovy` filetype only).
`:echo &filetype` in a Jenkinsfile → `Jenkinsfile` (must NOT be `groovy`).

## Troubleshooting (symptom → cause → fix)

| symptom | likely cause | fix |
|---|---|---|
| `<leader>gd` → "no doc for X" | gdoc index missing/stale | `lsp-gdoc --build` (or `:GdocBuild`) |
| `<leader>gd` works for stdlib but not a plugin/lib | that jar's `-sources.jar` not downloaded | rerun `jenkins-libs.sh --sources --javadoc --ln` (or `groovy-libs.sh --with-libs`), then `lsp-gdoc --build` |
| `lsp-gdoc --build` reports `sources: 0 jars` | no `-sources.jar` under the roots, or symlink not followed | run the download scripts first; `find -L` already follows `/opt/groovy/latest` |
| Jenkinsfile `gh.cliRun` completes but **no javadoc float** | `.tags` stale/missing, or method has no `/** */` above it | regenerate `.tags` via `gctags`; confirm the source has a javadoc block |
| Jenkinsfile `gh.` gives nothing (`[GT]`) | `.tags` missing, or source not `vars/<name>.groovy` | `gctags`; check `tagfiles()` includes the repo `.tags` |
| Jenkinsfile shows `unable to resolve class Library` | jenkinsfile got mapped to the groovy LSP | ensure `g:coc_filetype_map` = `{ 'yaml.ansible': 'ansible' }` (no Jenkinsfile), `:CocRestart` |
| completion **eats the word after the cursor** | `suggest.insertMode` back to `replace` | set `"suggest.insertMode": "insert"`, `:CocRestart` |
| vars/*.groovy: no LSP completion/hover | coc-groovy not attached / bad classpath | `:CocList services`; check `groovy.java.home` + referencedLibraries paths exist; `:CocRestart` |
| auto-pair fails before `/` or `"` | lexima suppress regex too strict | `~/.marslo/vimrc.d/extension`: negated char classes must include `/` and `"'\`` |
| download script: `declare -A: invalid option` / `local -n` error | bash < 4.3 (stock macOS 3.2) | run via homebrew bash |
| download script: `mkdir: /opt/...: Permission denied` | `/opt` not writable | `-p ~/jenkins` / `-p ~/groovy`, or sudo |
| `--with-libs`: `skip unknown group: <artifact>` | new bundled jar not in the group map | add `[artifact]='group'` to `LIB_GROUP` (verify sources 200 first) |
| `groovy-libs.sh` downloads wrong version | picked latest instead of system | omit `--latest`/`--pre`; check `detectGroovyHome` finds the right `lib/groovy-*.jar` |
| `--cleanup` deleted too much / too little | surgical globs only hit `[0-9]*`,`latest`,`plugins`,`extensions`,`logs` | preview first: `jenkins-libs.sh --cleanup --dryrun` |

## coc-settings.json — key entries

```jsonc
"groovy.project.referencedLibraries": [
  "/opt/homebrew/opt/groovy/libexec/lib/*",   // system (brew) groovy — bin jars
  "/opt/groovy/latest/*",                       // groovy-libs.sh downloads (sources/javadoc)
  "/opt/jenkins/latest/WEB-INF/lib/*",          // jenkins war libs
  "/Users/marslo/.groovy/lib/*"                 // jenkins-libs.sh --ln consolidates here
],
"suggest.insertMode": "insert"                  // default 'replace' eats the word after cursor
```

## ctags (Jenkinsfile + groovy shared-libs)

- `~/.ctags.d/groovy.ctags`: universal-ctags **groovy langdef** (`--langdef=groovy`, regexes
  for class/interface/trait/enum/constant/method → kinds `c/i/t/e/C/m/u/n/o/b`).
- `gctags` (in `~/.marslo/bin/ifunc.sh`) generates the repo `.tags` at the shared-libs root
  (`~/iMarslo/job/code/re/release-engineering/.tags`).
- `groovy_tags.vim` coc source: on `.` in groovy/Jenkinsfile, matches tag lines for
  `vars/<receiver>.groovy`, and — the enhancement — reads the source file at the tag's line,
  extracts the `/** ... */` javadoc immediately above the def, cleans it (`{@code}`→backticks,
  `{@link}`→name, strip HTML/entities), and sets it as the item's `info` (coc doc float).
  Filetypes: `['groovy','Jenkinsfile','jenkinsfile']`, shortcut `GT`, triggerOnly on `.`.

## gdoc — offline javadoc index for any classpath symbol

`~/.marslo/bin/lsp-gdoc`:
- `lsp-gdoc --build [DIR ...]` — for each `*-sources.jar` under the roots: `unzip` the sources into
  `~/.cache/nvim/gdoc/src/`, run `ctags -R --fields=+n --extras=+q` → `.tags` (with `line:`),
  and build `javadoc-map.tsv` (`FQN \t *-javadoc.jar \t Class.html`) from `*-javadoc.jar`.
- `lsp-gdoc --clean` — wipe `~/.cache/nvim/gdoc` (reports freed size; no-op if already absent).
- Default roots: `/opt/groovy/latest`, `~/.groovy/lib`, `/opt/jenkins/latest/WEB-INF/lib`
  (uses `find -L` to follow the `/opt/groovy/latest` symlink).

`~/.config/nvim/autoload/gdoc.vim`:
- `gdoc#hover()` — word under cursor (+ `Class.` qualifier if present) → look up in gdoc `.tags`
  (prefer class match) → read source at `line:` → extract javadoc above the def → float.
  Class-name fallback: render `Class.html` from a `*-javadoc.jar` via `unzip -p | pandoc -f html -t plain`.
- `gdoc#build()` — async `jobstart` of `lsp-gdoc --build`.
- Bound to `<leader>gd`; `:GdocBuild` command.

> [!NOTE]
> gdoc covers K-hover (point at a symbol) and `Class.method` reliably. It does NOT do `.`
> completion for arbitrary typed vars — that needs the LSP's type resolution, which
> groovy-language-server doesn't expose for javadoc. Javadoc is never in binary `.class`
> files, so bin-only jars need their `-sources.jar` / `-javadoc.jar` downloaded (that is what
> the two download scripts do).

## /opt/groovy/groovy-libs.sh

Download groovy core + libs + extensions with sources/javadoc into `/opt/groovy` (`-p` overrides).

| option | effect |
|---|---|
| `--groovy [VER]` | download groovy. no VER → **system groovy version** (auto-detected); fallback `5.0.8` |
| `--with-libs` (was `--groovy-libs`, deprecated) | scan the **system groovy lib** and fetch `-sources/-javadoc` for **every** jar at its own version/group |
| `--with-bin` | also fetch the compiled `.jar` (default: sources/javadoc only) |
| `--latest` | resolve maven's latest stable instead of the system version |
| `--pre` | resolve newest incl. alpha/beta/rc |
| `-e ARTIFACT` | extra maven extension (default: `groovy-cps jenkins-core`) |
| `--cleanup` | remove downloaded version dirs + `latest` + `extensions` under `-p`, then exit (keeps the script) |

Key internals:
- **System groovy home** auto-detected cross-platform (`detectGroovyHome`): `$GROOVY_HOME` →
  brew (`/opt/homebrew/opt/groovy/libexec`, `/usr/local/...`) → sdkman
  (`~/.sdkman/candidates/groovy/current`) → apt (`/usr/share/groovy`) → derived from the
  `groovy` launcher on PATH. Picks the first dir whose `lib/` has `groovy-*.jar`.
- **`--with-libs` needs the group per jar.** Most bundled jars have **no** `pom.properties`
  and maven search is rate-limited, so a **verified group map** is used (no runtime pom parse):
  prefix rules `groovy-*`→`org.apache.groovy`, `ant*`→`org.apache.ant`, `jline-*`/`jansi`→`org.jline`,
  `jackson-*`→`com.fasterxml.jackson.core`, `jackson-dataformat-*`→`…dataformat`,
  `junit-jupiter-*`→`org.junit.jupiter`, `junit-platform-*`→`org.junit.platform`; plus a
  `LIB_GROUP` table for singletons (testng, ivy, jna, jcommander→`org.jcommander`, gpars,
  qdox, xstream, snakeyaml, opentest4j, jsr166y→`org.codehaus.jsr166-mirror`, …). Unmapped jar
  → skip + warn "add to LIB_GROUP".
- **No system groovy** → fall back to a hardcoded `GROOVY_MODULES` list at the resolved
  version (org.apache.groovy); third-party deps can't be enumerated without the install.
- Version resolution is **deferred to main()** (sentinel `@default`) so `--latest`/`--pre`
  work regardless of flag order.

## /opt/jenkins/jenkins-libs.sh

Download jenkins war + a fixed plugin set + jenkins-core + cloudbees/maven extensions into
`/opt/jenkins` (`-p` overrides), maintaining `latest` symlinks.

| option | effect |
|---|---|
| `--lts` | latest LTS war (default: weekly) |
| `--ln` | refresh `~/.groovy/lib`: prune dangling links, then symlink war libs + core + plugin + extension jars |
| `--sources` | also fetch `-sources.jar` for war + plugin `WEB-INF/lib` jars (binary-only otherwise) |
| `--javadoc` | with `--sources`: also `-javadoc.jar` (implies `--sources`) |
| `--cleanup` | remove `~/.groovy/lib` links into `${JENKINS_ROOT}` + downloaded trees (version/latest/plugins/extensions/logs), then exit (respects `--dryrun`, keeps the script) |
| `--dryrun` | print planned actions only |
| `-P PLUGIN` / `-e ARTIFACT` / `-p PATH` | extra plugin / extension / install root |

Key internals:
- **Bundled-jar docs** (`fetchLibDocs`): war/plugin `WEB-INF/lib/*.jar` are unpacked from the
  archive and are binary-only. `resolveGav` reads `META-INF/maven/*/pom.properties` **matched by
  artifactId** (uber/shaded jars embed several poms). **pom-only / no network** — a jar without a
  matching pom is left unresolved and **skipped** (no maven-search fallback; the search API was
  rate-limited and stalled the run). `repoForGroup` routes `org.jenkins-ci*`/`io.jenkins*`/
  `org.jvnet.hudson*` → repo.jenkins-ci.org, else maven central. Downloads `-sources`/`-javadoc`
  next to the bin jar → picked up by `--ln` + gdoc.
- **Quiet output**: a 404 (no docs published) prints a concise `skip: <a>-<v><type> (not found)`
  to the console; the real `curl: (22) …` + full URL go to the run's log file
  (`${JENKINS_ROOT}/logs/<ts>.log`) only. Summary line: `fetched N doc jar(s), skipped M (no coords)`.
- jenkins-core + extensions already ship with sources/javadoc from their maven repos.
- Verified run: 78 war libs → **47 -sources + 44 -javadoc** fetched (rest 404/no-coords, skipped),
  `~/.groovy/lib` → 183 jars / 50 sources, gdoc index → ~438k symbols.

## Cross-cutting decisions / gotchas

- **bash >= 4.3** guard at the top of both scripts (associative arrays + namerefs). stock macOS
  `/bin/bash` is 3.2 → run via homebrew bash.
- Both scripts live **inside** their install root (`/opt/jenkins/jenkins-libs.sh`,
  `/opt/groovy/groovy-libs.sh`) → `--cleanup` uses **surgical globs** (`[0-9]*`, `latest`,
  `plugins`/`extensions`/`logs`), never `rm -rf` the whole root, so the script + user files survive.
- `/opt/{jenkins,groovy}` may need write perms on Linux → use `-p ~/...` or sudo.
- kubecolor/fzf/read-prompt tips live in `jenkins-up.sh` work, not here.
- lexima: `~/.marslo/vimrc.d/extension` suppress regexes allow auto-pairing before `/` and quotes
  (added `/` and `"'\`` to the negated char classes) — unrelated to the doc setup but same session.
