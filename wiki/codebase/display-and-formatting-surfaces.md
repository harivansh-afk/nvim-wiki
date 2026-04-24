# Display and formatting surfaces

> Sources: [Statusline mini-language parser](../tasks/19-statusline-parser.md)
> Updated: 2026-04-23

## Overview

This area covers the statusline-like mini-language surfaces where user-provided format strings are validated, rendered, truncated, and combined with display-state bookkeeping. Even though this surface looks cosmetic, it has dense parser logic, recursive evaluation, and a real history of memory-safety bugs.

## Covered task briefs

- [Statusline mini-language parser](../tasks/19-statusline-parser.md)

## Key files and entrypoints

- [src/nvim/statusline.c](../../raw/neovim/src/nvim/statusline.c)
- [src/nvim/optionstr.c](../../raw/neovim/src/nvim/optionstr.c)
- Supporting docs:
  - [runtime/doc/options.txt](../../raw/neovim/runtime/doc/options.txt)
  - [runtime/doc/vim_diff.txt](../../raw/neovim/runtime/doc/vim_diff.txt)

## Primary docs and knobs

- High-value knobs: `'statusline'`, `'statuscolumn'`, `'rulerformat'`, `'fillchars'`, `'laststatus'`
- The important relationship here is validator versus renderer, not just option text versus display output.

## How agents should navigate this area

1. Compare `check_stl_option()` with `build_stl_str_hl()` before focusing on any single format feature.
2. Separate plain format parsing from recursive evaluation features such as `%!` and `%{...}`.
3. Track byte counts, cell widths, truncation, and click-label bookkeeping independently. They are distinct state machines.
4. If the issue is actually a more general Ex or option-text parser problem, continue with [Tag and command surfaces](tag-and-command-surfaces.md) or [Trust and local config surfaces](trust-and-local-config-surfaces.md).

## See Also

- [Tag and command surfaces](tag-and-command-surfaces.md)
- [Trust and local config surfaces](trust-and-local-config-surfaces.md)
- [Task brief navigation](../agents/task-brief-navigation.md)
