# Treesitter language loader

## Overview

Treesitter's language resolver searches `runtimepath` for `parser/<lang>.so` and `dlopen`s it. [GHSA-6f9m](../cves/GHSA-6f9m.md) patched the obvious path-traversal in injection language names, but the broader surface remains: any code path that feeds an attacker-influenced string into `vim.treesitter.language.add()` or its underlying loader can load an attacker-placed `.so`. Maintainers track the long-term fix as WASM sandboxing ([#23579](https://github.com/neovim/neovim/issues/23579)); until then, every name-validation site is the policy gate.

## Relevant options and knobs

- `'runtimepath'`: search prefix for `parser/<lang>.so`. Any plugin or user write to runtimepath becomes a parser path.
- `vim.treesitter.language.add(lang, {path = ...})`: explicit-path API — bypasses runtimepath.
- Injection captures (`@injection.language`) inside query files: untrusted node text becomes a language name.

## Relevant files

- [`runtime/lua/vim/treesitter/language.lua`](../../raw/neovim/runtime/lua/vim/treesitter/language.lua) (`add`, `resolve`)
- [`runtime/lua/vim/treesitter/languagetree.lua`](../../raw/neovim/runtime/lua/vim/treesitter/languagetree.lua) (injection resolution)
- [`src/nvim/lua/treesitter.c`](../../raw/neovim/src/nvim/lua/treesitter.c) (`tslua_add_language`)

## File tree

```text
runtime/lua/vim/treesitter/
  language.lua
  languagetree.lua
  query.lua
src/nvim/lua/
  treesitter.c
```

## Big-picture references

- [`runtime/doc/treesitter.txt`](../../raw/neovim/runtime/doc/treesitter.txt)
- [GHSA-6f9m-hj8h-xjgj](https://github.com/neovim/neovim/security/advisories/GHSA-6f9m-hj8h-xjgj): the patched language-name path-traversal.

## Recent fix / history signal

- Fix commits for GHSA-6f9m: [`8b11cf5`](https://github.com/neovim/neovim/commit/8b11cf5092e0dfe45ee0f9cf0b72ce4ddeb8740c), [`1789ce1`](https://github.com/neovim/neovim/commit/1789ce1195757651584528c16c7abf4c92381846).
- [#38132](https://github.com/neovim/neovim/issues/38132): `resolve_lang` rejects hyphenated language names — cross-check this regex for bypasses.
- [#38985 §2.3](https://github.com/neovim/neovim/issues/38985): "malicious treesitter parsers? Mitigation: WASM?" — open.

## Audit focus

- Variant-hunt the GHSA-6f9m fix. Every place a language name reaches `nvim__get_runtime` or `dlopen` should be checked — including `add()`'s `path = ...` branch, queries, and async loads.
- Trace `@injection.language` capture text through `languagetree.lua` to confirm the name passes through `resolve_lang`'s validator and not a different code path.
- Look at parser cache invalidation: if a malicious `.so` is loaded once and cached, does runtimepath demotion clear it?

## See Also

- [GHSA-6f9m (accepted)](../cves/GHSA-6f9m.md)
- [Reachability §treesitter](../reachability.md)
