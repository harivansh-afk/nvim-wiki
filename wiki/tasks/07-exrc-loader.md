# Project-local exrc loader

## Overview

This is a Neovim-specific project-local configuration boundary. [`src/nvim/main.c`](../../../neovim/src/nvim/main.c) enters `do_exrc_initialization()`, which loads [`runtime/lua/vim/_core/exrc.lua`](../../../neovim/runtime/lua/vim/_core/exrc.lua); that module searches upward for `.nvim.lua`, `.nvimrc`, and `.exrc`, then executes trusted files. It ranks highly because this is an intentional arbitrary-code execution feature, and the safety of the feature depends on search order, stop conditions, and trust enforcement being exactly right.

## Relevant options and knobs

- `'exrc'`: master switch for project-local configuration loading.
- The effective search scope is “current directory plus parents”, which is a functional knob even though it is not represented as a separate option.
- Lua and Vimscript exrc files take different execution paths (`loadstring()` vs `nvim_exec2()`), which matters operationally even though it is not a user option.

## Relevant files

- [`src/nvim/main.c`](../../../neovim/src/nvim/main.c)
- [`runtime/lua/vim/_core/exrc.lua`](../../../neovim/runtime/lua/vim/_core/exrc.lua)
- [`src/nvim/runtime.c`](../../../neovim/src/nvim/runtime.c)

## File tree

```text
src/
  nvim/
    main.c
    runtime.c
runtime/
  lua/
    vim/
      _core/
        exrc.lua
```

## Big-picture references

- [`runtime/doc/options.txt`](../../../neovim/runtime/doc/options.txt): authoritative `exrc` behavior and trust requirements.
- [`runtime/doc/starting.txt`](../../../neovim/runtime/doc/starting.txt): startup order, including when local config is discovered and sourced.
- [`runtime/doc/vim_diff.txt`](../../../neovim/runtime/doc/vim_diff.txt): explains the Nvim-specific expansion of `exrc` to `.nvim.lua` and trust prompts.

## Recent fix / history signal

- Nvim 0.12 docs changed `exrc` to search parent directories and clarified that unsetting `exrc` should stop further search.
- Docs also emphasize that trusted `exrc` replaced the older “secure option” model, so policy mistakes here have outsized impact.

## Audit focus

- Review upward-search ordering and the “stop searching if current exrc unsets `exrc`” behavior.
- Check Lua vs Vimscript execution paths for mismatched context attribution, sandbox semantics, and error reporting.
- Verify that trust checks happen on the normalized real path actually being executed.
- Treat parent-directory search as a distinct risk multiplier during review, not just a convenience feature.

## See Also

- [Trust database and :trust bridge](08-trust-db.md)
- [Modeline execution pipeline](05-modelines.md)
- [mapset secure-mode gate](06-mapset-secure.md)
