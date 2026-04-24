# Lua `require` from CWD

## Overview

This is a default-on auto-execution surface flagged by core maintainers as in-scope ([#38985 §1.4](https://github.com/neovim/neovim/issues/38985), [#38966](https://github.com/neovim/neovim/issues/38966)). Neovim overrides Lua's loader via [`runtime/lua/vim/_init_packages.lua`](../../raw/neovim/runtime/lua/vim/_init_packages.lua) (`vim._load_package`), but stock `package.path` and `package.cpath` still include `./?.lua` and `./?.so` as fallbacks. Any `require('foo')` whose name doesn't resolve via `runtimepath` falls through to CWD, executing whatever an attacker placed there.

## Relevant options and knobs

- `package.path`, `package.cpath`: the inherited Lua loader chain — `./?.lua` first.
- `'runtimepath'`: searched before `package.path` by `vim._load_package`; if a hit exists here, CWD is skipped.
- `nvim -l`: standalone Lua mode where `./?.lua` is intentional.

## Relevant files

- [`runtime/lua/vim/_init_packages.lua`](../../raw/neovim/runtime/lua/vim/_init_packages.lua)
- [`runtime/lua/vim/loader.lua`](../../raw/neovim/runtime/lua/vim/loader.lua)
- [`src/nvim/lua/executor.c`](../../raw/neovim/src/nvim/lua/executor.c)

## File tree

```text
runtime/lua/vim/
  _init_packages.lua
  loader.lua
src/nvim/lua/
  executor.c
```

## Big-picture references

- [`runtime/doc/lua.txt`](../../raw/neovim/runtime/doc/lua.txt): documents `vim._load_package` and stdlib package model.
- [`runtime/doc/lua-plugin.txt`](../../raw/neovim/runtime/doc/lua-plugin.txt): plugin loader semantics.

## Recent fix / history signal

- [#38966](https://github.com/neovim/neovim/issues/38966): tracking issue, justinmk: *"if by default `require('foo')` executes `foo.lua` from CWD, then yes we need to plug that."*
- [#38985 §1.4](https://github.com/neovim/neovim/issues/38985): listed as ⚠️ TODO in the maintainer threat model.

## Audit focus

- Find a `pcall(require, '<name>')` in a default-shipped runtime file (or core-recommended plugin pattern) whose name doesn't exist in `runtimepath`. That's the trigger.
- Verify whether `vim._load_package` actually catches the lookup before `package.path` for every code path — check both `require()` directly and indirect callers (`loadfile`, `dofile`, `loadstring`).
- Look at `nvim -l` interactions: maintainers want CWD on for `-l` but off for normal startup. Find the divergence point.

## See Also

- [Project-local exrc loader](07-exrc-loader.md)
- [Trust database and :trust bridge](08-trust-db.md)
- [Reachability §threat model](../reachability.md#threat-model-justinmk-38985)
