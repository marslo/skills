# Regression since `6c1cf016`: pyright completion items lose `documentation`/`detail` — advertised `completionList.itemDefaults.data` + `applyKind` are not preserved, breaking `completionItem/resolve`

> [!IMPORTANT]
> Since commit `6c1cf016`, LSP completion items from `coc-pyright` no longer show
> `documentation`/`detail` in the popup-menu float. `completionItem/resolve` returns
> the item unchanged. Rolling back to the parent commit `2abe7008` restores it.

## Summary

`6c1cf016` started advertising the LSP 3.18 completion capabilities `textDocument.completion.completionList.itemDefaults` (now including `"data"`) and `applyKindSupport: true`.

With those advertised, **pyright hoists the request-shared `data` (`{ uri, position }`) into `CompletionList.itemDefaults.data`**, leaves only `{ symbolLabel }` on each item, and sends `applyKind = { data: 2 /* Merge */ }` telling the client to merge them back.

coc **does not preserve `applyKind` nor `itemDefaults.data`** when it processes the response. `applyItemDefaults()` is therefore skipped, each item keeps `data = { symbolLabel }` (no `uri`/`position`), and pyright's `completionItem/resolve` can no longer locate the symbol — so it returns no `documentation`/`detail`.

`textDocument/hover` is unaffected (independent code path), which proves pyright *has* the documentation; only the completion-resolve round-trip is broken.

## Environment

| Component         | Version                                       |
| ----------------- | --------------------------------------------- |
| coc.nvim          | `release` @ `1245b4a7` (`v0.0.82-337`)        |
| Regression commit | `6c1cf016` (2026-08-04)                       |
| Last good commit  | `2abe7008` (2026-08-01, parent of `6c1cf016`) |
| coc-pyright       | `1.1.411` (bundled pyright `1.1.411`)         |
| Neovim            | `0.12.4`                                      |
| Node.js           | `v26.7.0`                                     |

## Symptom

- LSP (`[LS]`) completion items: **no** documentation/detail float.
- Snippet (`[S]`) items: documentation still shows (they carry it inline, no resolve needed).
- Before `6c1cf016`, `[LS]` items showed the resolved signature/docstring float.

*(screenshots attached separately)*

## Root cause analysis

### The LSP mechanism (pyright side)

pyright uses **LSP 3.17 `CompletionList.itemDefaults` + 3.18 `applyKind`** when the client advertises support. For a completion request it returns:

```jsonc
{
  "isIncomplete": false,
  "itemDefaults": { "data": { "uri": "file://…/a.py", "position": { "line": 3, "character": 3 } } },
  "applyKind":    { "data": 2 },          // 2 = ApplyKind.Merge
  "items": [
    { "label": "getcwd", "kind": 3, "sortText": "11.9999.getcwd",
      "data": { "symbolLabel": "getcwd" } }   // per-item data is ONLY the symbol label
  ]
}
```

The client is required to **merge** `itemDefaults.data` into each `item.data` (`applyKind.data === Merge`), yielding `{ uri, position, symbolLabel }`. pyright's `resolveCompletionItem` relies on `data.uri` + `data.position` to re-locate the symbol and attach `documentation`/`detail`.

### What `6c1cf016` changed (coc side)

The commit added, to the advertised client capabilities
(`textDocument.completion.completionList`):

```js
itemDefaults: ["commitCharacters", "editRange", "insertTextFormat", "insertTextMode", "data"],
applyKindSupport: true
```

This is what makes pyright hoist `data` into `itemDefaults.data` (as above). Before the commit, coc did not advertise `completionList.itemDefaults`, so pyright inlined the full `{ uri, position, symbolLabel }` into every item's `data` and resolve worked.

### Where it breaks

coc **drops `applyKind` and `itemDefaults.data`** from the response before the merge step (the protocol→code conversion of the completion result does not carry them). In `CompletionItemFeature.provideCompletionItems`:

```js
let itemDefaults = this.itemDefaults = toObject(result["itemDefaults"]);
let applyKind = isCompletionList(result) ? result["applyKind"] : void 0;
// guard is FALSE at runtime → applyItemDefaults() is never called
if (applyKind || itemDefaults.data != null)
  for (let item of completeItems) applyItemDefaults(item, itemDefaults, applyKind);
```

Instrumented at runtime for a pyright completion:

```
isCompletionList(result) = true
result.applyKind         = undefined     // dropped
itemDefaults.data        = undefined     // dropped   (only editRange/etc. survive)
ApplyKind.Merge          = 2
item.data BEFORE = {"symbolLabel":"getcwd"}
item.data AFTER  = {"symbolLabel":"getcwd"}   // unchanged: guard was false, merge skipped
```

So each item is sent to `completionItem/resolve` with `data = { symbolLabel }` only.
pyright cannot resolve the symbol and returns the item verbatim (no `documentation`).

> [!NOTE]
> `applyItemDefaults()` itself already handles `ApplyKind.Merge` correctly
> (`item.data = Object.assign({}, itemDefaults.data, item.data)`). It is **dead code at
> runtime** because its inputs (`applyKind`, `itemDefaults.data`) are stripped before it runs.

## Reproduction (headless)

### 0. Versions

```bash
git -C ~/.vim/plugged/coc.nvim describe --tags
node --version
nvim --version | head -1
```

### 1. Bisect — parent commit works, regression commit doesn't

```bash
cd ~/.vim/plugged/coc.nvim
git log --oneline -S 'applyKindSupport' -- build/index.js   # -> introduced by 6c1cf016
# test parent (last good):
git checkout 2abe7008        # documentation shows again
# restore:
git checkout release
```

### 2. Prove pyright hoists `data` only when the client advertises the capability (bypasses coc)

Raw LSP client (Node, no deps) talking straight to the bundled `pyright-langserver`. Toggle the `completionList` capability block to see the difference.

```js
// raw-lsp.js  —  node raw-lsp.js
const { spawn } = require('child_process');
const SERVER = require('os').homedir() +
  '/.config/coc/extensions/node_modules/coc-pyright/node_modules/pyright/dist/pyright-langserver.js';
const FILE = '/tmp/cocbug/repro.py';                     // contains:  import os\nos.getcwd()\n
const uri = 'file://' + FILE, text = require('fs').readFileSync(FILE, 'utf8');
const srv = spawn(process.execPath, [SERVER, '--stdio']);
let buf = Buffer.alloc(0); const pend = new Map(); let id = 0;
srv.stdout.on('data', d => { buf = Buffer.concat([buf, d]); for (;;) {
  const h = buf.indexOf('\r\n\r\n'); if (h < 0) break;
  const len = +/Content-Length: (\d+)/i.exec(buf.slice(0, h))[1];
  if (buf.length < h + 4 + len) break;
  const m = JSON.parse(buf.slice(h + 4, h + 4 + len)); buf = buf.slice(h + 4 + len);
  if (pend.has(m.id)) { pend.get(m.id)(m); pend.delete(m.id); } } });
const send = o => { const s = JSON.stringify(o);
  srv.stdin.write('Content-Length: ' + Buffer.byteLength(s) + '\r\n\r\n' + s); };
const req = (method, params) => new Promise(r => { const i = ++id; pend.set(i, r);
  send({ jsonrpc: '2.0', id: i, method, params }); });
const sleep = ms => new Promise(r => setTimeout(r, ms));
(async () => {
  await req('initialize', { processId: process.pid, rootUri: 'file:///tmp/cocbug',
    capabilities: { textDocument: { completion: { completionItem: {
      documentationFormat: ['markdown', 'plaintext'],
      resolveSupport: { properties: ['documentation', 'detail', 'additionalTextEdits'] } },
      // ← comment the next line out to reproduce the pre-6c1cf016 (working) behavior
      completionList: { itemDefaults: ['data'], applyKindSupport: true } } } } });
  send({ jsonrpc: '2.0', method: 'initialized', params: {} });
  send({ jsonrpc: '2.0', method: 'textDocument/didOpen',
    params: { textDocument: { uri, languageId: 'python', version: 1, text } } });
  await sleep(2500);
  const c = await req('textDocument/completion', { textDocument: { uri }, position: { line: 1, character: 3 } });
  console.log('itemDefaults =', JSON.stringify(c.result.itemDefaults));
  console.log('applyKind    =', JSON.stringify(c.result.applyKind));
  const it = c.result.items.find(i => i.label === 'getcwd');
  console.log('item.data    =', JSON.stringify(it.data));
  const r = await req('completionItem/resolve', it);
  console.log('resolved has documentation =', 'documentation' in r.result);
  srv.kill(); process.exit(0);
})();
```

| capability advertised                                             | `item.data` returned                                            | resolve has documentation         |
| ----------------------------------------------------------------- | --------------------------------------------------------------- | --------------------------------- |
| **with** `completionList.itemDefaults:['data'], applyKindSupport` | `{symbolLabel}` (+ `itemDefaults.data`/`applyKind` on the list) | **true** — *if the client merges* |
| **without** it (pre-`6c1cf016`)                                   | `{uri, position, symbolLabel}`                                  | **true**                          |

The point: pyright behaves correctly either way. The client must merge when it opts in.

### 3. Prove coc drops it and resolve fails (through coc)

```bash
mkdir -p /tmp/cocbug && printf 'import os\nos.getcwd()\n' > /tmp/cocbug/repro.py
printf '{ "python.analysis.typeCheckingMode":"basic" }\n' > /tmp/cocbug/coc-settings.json
```

```vim
" /tmp/cocbug/repro.vim
let g:coc_config_home = '/tmp/cocbug'
let s:o = '/tmp/cocbug/out.txt' | call writefile([], s:o)
function! s:R(...) abort
  if !coc#rpc#ready() | call timer_start(500, function('s:R')) | return | endif
  let l:running = 0
  for s in CocAction('services') | if s.id =~? 'pyright' && s.state ==? 'running' | let l:running = 1 | endif | endfor
  if !l:running | call timer_start(500, function('s:R')) | return | endif
  let u = 'file://' . expand('%:p')
  let res = CocRequest('pyright', 'textDocument/completion', {'textDocument':{'uri':u},'position':{'line':1,'character':3}})
  let items = type(res) == v:t_dict ? res.items : res
  for it in items | if get(it,'label','') ==# 'getcwd'
      call writefile(['item.data = ' . json_encode(it.data)], s:o, 'a')
      let r = CocRequest('pyright', 'completionItem/resolve', it)
      call writefile(['resolve has documentation = ' . has_key(r, 'documentation')], s:o, 'a')
      break | endif | endfor
  qa!
endfunction
autocmd VimEnter * call timer_start(1500, function('s:R'))
```

```bash
nvim --headless /tmp/cocbug/repro.py -S /tmp/cocbug/repro.vim ; cat /tmp/cocbug/out.txt
```

| coc version | `item.data` (raw response) | resolve has documentation |
|---|---|---|
| `2abe7008` (parent, **expected**) | `{"uri":…,"position":…,"symbolLabel":"getcwd"}` | `1` ✅ |
| `1245b4a7` (`6c1cf016`, **actual**) | `{"symbolLabel":"getcwd"}` | `0` ❌ |

> [!NOTE]
> `CocRequest` returns the raw wire response (before `applyItemDefaults`). At `2abe7008` coc
> does not advertise `itemDefaults`, so pyright inlines the full `data` — hence the raw
> per-item `data` already differs between the two versions. The end-user symptom (missing
> float) and the runtime instrumentation in the root-cause section confirm the same for the
> real completion pipeline.

## Fix approach

**Preferred (proper fix).** Preserve `CompletionList.applyKind` and `itemDefaults.data`
through the completion-response conversion so that `applyItemDefaults()` actually receives
them. The existing merge already handles `ApplyKind.Merge`
(`item.data = Object.assign({}, itemDefaults.data, item.data)`); it just needs its inputs.
Concretely: the protocol→code conversion of the completion result should not strip
`applyKind` / `itemDefaults.data` (LSP 3.18), and the guard
`if (applyKind || itemDefaults.data != null)` should see them.

**Interim workaround (local, verified).** If preserving them is not trivial, stop advertising the `data` item-default so servers inline per-item `data` (pre-`6c1cf016` behavior):

```diff
             "insertTextFormat",
-            "insertTextMode",
-            "data"
+            "insertTextMode"
           ],
-          applyKindSupport: true
+          applyKindSupport: false
```

With this change `completionItem/resolve` returns `documentation`/`detail` again

verified headless with coc-pyright + pyright 1.1.411, Neovim 0.12.4, Node v26.7.0.
