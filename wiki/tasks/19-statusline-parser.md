# Statusline mini-language parser

## Overview

This is a mixed inherited-and-Nvim-extended mini-language parser. [`src/nvim/statusline.c`](../../../neovim/src/nvim/statusline.c) renders `statusline` and `statuscolumn` format strings in `build_stl_str_hl()`, while [`src/nvim/optionstr.c`](../../../neovim/src/nvim/optionstr.c) validates them with `check_stl_option()`. It ranks highly because the code is parser-dense, historically overflow-prone, and mixes byte counts, display widths, recursive evaluation, and click-target bookkeeping.

## Relevant options and knobs

- `'statusline'`: primary format string surface.
- `'statuscolumn'`: uses closely related formatting logic but has its own display semantics.
- `'rulerformat'`: another statusline-like format string that exercises related code paths.
- `'fillchars'` and `'laststatus'`: not parsers themselves, but they materially change rendering assumptions around the same subsystem.

## Relevant files

- [`src/nvim/statusline.c`](../../../neovim/src/nvim/statusline.c)
- [`src/nvim/optionstr.c`](../../../neovim/src/nvim/optionstr.c)

## File tree

```text
src/
  nvim/
    statusline.c
    optionstr.c
```

## Big-picture references

- [`runtime/doc/options.txt`](../../../neovim/runtime/doc/options.txt): authoritative documentation for `statusline`, `statuscolumn`, and related formatting options.
- [`runtime/doc/vim_diff.txt`](../../../neovim/runtime/doc/vim_diff.txt): documents Nvim-specific statusline/statuscolumn changes and format-surface growth.
- [`runtime/doc/dev_arch.txt`](../../../neovim/runtime/doc/dev_arch.txt): useful because option definitions are generated from metadata, while statusline rendering itself remains hand-coded.

## Recent fix / history signal

- `a416494e64` / `b9459fba26`: security fix for stack-buffer-overflow in `build_stl_str_hl()` (#38102).
- `d672f0f494` / `e0eb967f8a`: fix for potential buffer underrun when setting statusline-like options (#39063).
- Older history also includes invalid-memory-access and overflow fixes for bad statusline values, showing a long-lived bug class rather than a one-off regression.

## Audit focus

- Review validator/renderer mismatches: `check_stl_option()` being more permissive than `build_stl_str_hl()` is exactly the kind of drift that creates bugs.
- Check every place where byte counts, cell widths, and moved/truncated substrings interact.
- Treat recursive evaluation via `%!` and `%{...}` as a separate risk layer on top of plain format parsing.
- Audit click-label, group-depth, and truncation bookkeeping as stateful parser logic, not display polish.

## See Also

- [Ex address parser](20-ex-address-parser.md)
- [Modeline execution pipeline](05-modelines.md)
