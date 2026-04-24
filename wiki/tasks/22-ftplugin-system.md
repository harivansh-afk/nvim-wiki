# Ftplugin shell sinks via `'filetype'`

## Overview

This is the Tier-A surface justinmk explicitly named: *"if a ftplugin invokes `system()`, a malicious filename placed in the right location can invoke shell code. And this has happened more than once"* ([#39044](https://github.com/neovim/neovim/issues/39044)). Modeline `vim:set ft=<x>` accepts any name validated by `valid_filetype()`, then triggers `FileType` autocmds and sources `runtime/ftplugin/<x>.{vim,lua}` and `runtime/indent/<x>.vim`. Any of those that invoke `system()`, `jobstart()`, or `:!` against `expand('%')` or other attacker-controlled values is an exploitable chain.

## Relevant options and knobs

- `'filetype'`, `'syntax'`: modeline-settable; trigger respective autocmds.
- `'modeline'`, `'modelines'`: default on; controls whether the trigger fires.
- `loadplugins`, `'runtimepath'`: scope of which ftplugins are sourced.

## Relevant files

- [`runtime/lua/vim/filetype.lua`](../../raw/neovim/runtime/lua/vim/filetype.lua)
- [`src/nvim/runtime.c`](../../raw/neovim/src/nvim/runtime.c) (`source_runtime_vim_lua`)
- [`runtime/ftplugin/`](../../raw/neovim/runtime/ftplugin/) (every `.vim`/`.lua` here is a sink candidate)
- [`src/nvim/buffer.c`](../../raw/neovim/src/nvim/buffer.c) (modeline → option apply)

## File tree

```text
runtime/
  ftplugin/*.{vim,lua}
  indent/*.vim
  lua/vim/filetype.lua
src/nvim/
  runtime.c
  buffer.c
```

## Big-picture references

- [`runtime/doc/filetype.txt`](../../raw/neovim/runtime/doc/filetype.txt)
- [`runtime/doc/options.txt`](../../raw/neovim/runtime/doc/options.txt) `*'filetype'*`, modeline section.

## Recent fix / history signal

- [#39044](https://github.com/neovim/neovim/issues/39044): justinmk on why whitelist hardening of modeline doesn't help — `'filetype'` re-opens the surface.
- [#38985 §1.5](https://github.com/neovim/neovim/issues/38985): listed as ❓ open in the maintainer threat model.
- Sibling task [`06-mapset-secure`](06-mapset-secure.md) already documents the gate-drift on the modeline → `FileType` boundary.

## Audit focus

- Grep every shipped `runtime/ftplugin/*.{vim,lua}` for `system(`, `jobstart(`, `':!'`, `\`!`, `getcompletion(.*'shellcmd'` and the lua equivalents. Each hit is a candidate.
- Trace the argument flow back to `expand('%')`, `bufname()`, `b:current_compiler`, or any path-derived value an attacker can influence via the filename.
- Review `runtime/indent/<x>.vim` and `runtime/syntax/<x>.vim` siblings — same surface.
- Verify that `secure` is not (still) reset before the handler runs (cross-check [`06-mapset-secure`](06-mapset-secure.md)).

## See Also

- [Modeline execution pipeline](05-modelines.md)
- [`mapset` secure-mode gate](06-mapset-secure.md)
- [CVE: modelineexpr-callback (rejected)](../cves/modelineexpr-callback.md)
