# Runtimepath sourcing

## Overview

`'runtimepath'` is searched at startup and on demand for `*.vim` and `*.lua` to source. Anything that mutates `runtimepath` (plugin managers, `:set rtp+=`, `vim.opt.runtimepath:append`) directly extends Neovim's auto-execute scope. The surface is the chain *attacker-influenced path → `do_in_path` / `source_runtime_vim_lua` → `do_source` → arbitrary Lua/Vim execution* — without any `:trust` gate.

## Relevant options and knobs

- `'runtimepath'`, `'packpath'`: the search list. Untrusted writes here win.
- `vim.pack.add()`, `:packadd`: programmatic runtimepath extension.
- `XDG_CONFIG_HOME`, `XDG_DATA_HOME`: environment-driven defaults; CI/container envs may set these.

## Relevant files

- [`src/nvim/runtime.c`](../../raw/neovim/src/nvim/runtime.c) (`do_in_path`, `source_runtime`, `source_runtime_vim_lua`, `do_source`)
- [`src/nvim/option.c`](../../raw/neovim/src/nvim/option.c) (`runtimepath` set handlers)
- [`runtime/lua/vim/pack.lua`](../../raw/neovim/runtime/lua/vim/pack.lua) (programmatic adders)

## File tree

```text
src/nvim/
  runtime.c
  option.c
runtime/lua/vim/
  pack.lua
```

## Big-picture references

- [`runtime/doc/options.txt`](../../raw/neovim/runtime/doc/options.txt) `*'runtimepath'*`, `*'packpath'*`.
- [`runtime/doc/starting.txt`](../../raw/neovim/runtime/doc/starting.txt): startup-time sourcing order.

## Recent fix / history signal

- [#10732](https://github.com/neovim/neovim/issues/10732): "Security: Limit runtimepath evaluation". justinmk: *"Even without root that is risky. Why spend time implementing a half-measure?"* — closed without a fix.
- [#36037](https://github.com/neovim/neovim/issues/36037): `vim.pack.add` checks out the wrong commit when the pinned `version` is unavailable — supply-chain-adjacent.

## Audit focus

- Find every code path that writes `runtimepath` from a value an attacker can influence (env vars, parsed config, modeline-settable proxies, RPC). Each one is an injection point.
- Check `do_in_path` for path-traversal in the `name` argument when callers concatenate user input.
- Look at `:packadd`'s name validation — same shape as the treesitter language-name traversal.
- Race conditions: can a plugin manager be coerced into adding a path *before* `:trust` checks fire?

## See Also

- [Lua require from CWD](21-package-path-cwd.md)
- [Treesitter language loader](23-treesitter-language-loader.md)
- [Project-local exrc loader](07-exrc-loader.md)
