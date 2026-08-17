# lore-unity

Unity file-format chunker plugin for [Lore](https://github.com/wryskware/lore) —
the first out-of-tree consumer of Lore's declarative chunker-plugin contract.

A Lore plugin is data, not code: this repo ships a `lore-plugin.toml` manifest
plus tree-sitter grammars compiled to WebAssembly. Lore's own engine does all
chunking; nothing here executes.

| Format | Strategy |
|---|---|
| `.uxml` (UI Toolkit layouts) | XML grammar, element-level chunks |
| `.uss` (UI Toolkit styles) | CSS grammar, rule-level chunks |
| `.unity` / `.prefab` / `.asset` (serialized YAML) | size-capped line windows |

Status: scaffold — plugin authored against Lore's Phase 1 engine (in progress).
