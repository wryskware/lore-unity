# lore-unity

Unity file-format chunker plugin for [Lore](https://github.com/wryskware/lore) —
the first out-of-tree consumer of Lore's declarative chunker-plugin contract.

A Lore plugin is data, not code: this repo ships a `lore-plugin.toml` manifest
plus tree-sitter grammars compiled to WebAssembly. Lore's own engine does all
chunking; nothing here executes. The plugin cannot mint chunk IDs, build
embedding headers, or set authority metadata, because it never emits anything.

## What it covers

| Extension | Strategy | Shape |
|---|---|---|
| `.uxml` | XML grammar | one chunk per element, symbol path `ui:UXML.ui:VisualElement.ui:Label` |
| `.uss` | CSS grammar | one chunk per rule, symbol path is the selector (`.lex-card--active`) |
| `.unity` `.prefab` `.asset` | line windows | `language = yaml`, files over 256 KB skipped |
| `.shader` | line windows | `language = shaderlab` |

Not claimed: `.hlsl`, `.cginc`, and the HLSL inside a `.shader` block.
`tree-sitter-hlsl` publishes no `.wasm` release asset, so that would mean
building and vendoring a grammar — future work, out of v1 scope. `.cs` is
Lore's own built-in and is never claimable by a plugin.

## Install

```sh
git clone git@github.com:wryskware/lore-unity.git
lore plugin add ./lore-unity
```

`lore plugin add` copies this directory into the daemon's data dir. Then opt in
per project, because plugin routing is a repo-visible choice — in the Unity
project's `.lore.toml`:

```toml
[plugins]
enable = ["unity"]
```

Restart the daemon and re-index. Installing or editing a plugin changes its
fingerprint (a hash over the manifest and every `.wasm` it references), which
Lore folds into the per-file content hash, so exactly the files this plugin
owns get re-chunked.

Two checks worth running afterwards: `lore plugin list` should show `unity`
with both grammars available, and `lore status` reports files that fell back
because a named plugin was missing or its grammar would not load.

Lore must be built with its wasm-grammar feature, or the manifest parses but
both grammar chunkers report unavailable and their files take the fallback
path.

## Grammar provenance

Both artifacts are unmodified upstream release assets, carried over from the
Lore feasibility spike that verified them.

| File | Upstream | Release | ABI | Size |
|---|---|---|---|---|
| `xml.wasm` | [tree-sitter-grammars/tree-sitter-xml](https://github.com/tree-sitter-grammars/tree-sitter-xml) | v0.7.0 | 14 | 47032 B |
| `css.wasm` | [tree-sitter/tree-sitter-css](https://github.com/tree-sitter/tree-sitter-css) | v0.25.0 | 15 | 128668 B |

Lore's runtime accepts grammar ABI 13–15 and rejects anything else at load
with a typed reason rather than a panic.

## Mapping notes

The node-kind strings in `lore-plugin.toml` were derived by dumping the real
grammars over real files, not read off grammar documentation. Every kind the
manifest names is present in `fixtures/dumps/`. Running all 29 `.uxml` and 25
`.uss` files in Lexomancy through a port of Lore's walker produced 691 chunks,
zero parse errors, zero oversized chunks, and zero elements or rules that fell
back to naming themselves after their node kind.

Two things the engine's own toy fixture did not exercise:

- **`CharData` has to be an attachment.** In real XML the whitespace between a
  comment and the element it documents is its own node, so
  `attachments = ["Comment"]` alone never fires — the attach-leading walk stops
  at the first non-attachment sibling. Without `CharData`, a 2 KB design
  comment in `UnitFrame.uxml` detaches from its `ui:Instance` and becomes an
  anonymous `statements` chunk.
- **Self-closing elements use `EmptyElemTag`, not `STag`.** Both put the tag
  name in a `Name` child at the same depth, so `name_kinds = ["Name"]` covers
  both, but the `bodies = ["content"]` node exists only on the paired-tag form.

Known limits, both benign:

- A descendant selector is recorded with its whitespace collapsed, so `.a .b`
  becomes the symbol path `.a.b`. Searchable, not a faithful selector.
- tree-sitter-css 0.25 flags the modern media range syntax
  (`@media (width < 800px)`) as an error inside the query. Spans stay exact and
  the rules inside still chunk correctly.

## Fixtures

`fixtures/` holds nine files copied unmodified from Lexomancy — the author's
own Unity project — chosen to cover nested elements, template declarations and
instances, self-closing elements, XML prologs, attached and detached comments,
`:root` custom properties, and `-unity-*` properties.

`fixtures/dumps/` holds a named-node tree for one UXML and one USS fixture,
plus the chunk output the manifest produces for all nine. They are reference
material for anyone changing a mapping: change the manifest, re-dump, diff.

End-to-end verification against a running Lore daemon lives in the Lore repo,
not here.
