# Spell affix parser

## Overview

This is a mostly Vim-derived hostile-file parser and generator. [`src/nvim/spellfile.c`](../../../neovim/src/nvim/spellfile.c) reads external `.aff` and `.dic` data, tokenizes affix rules, and expands them into internal structures and generated spell words. It ranks highly because recent fixes included stack-buffer overflows in spell generation, and because this code mixes hand-rolled text parsing with fixed-size buffers and recursive word-building logic.

## Relevant options and knobs

- `'spellfile'`: controls custom spell file locations.
- `'spelllang'`: influences which spell resources are loaded and used.
- `::mkspell`: operationally important because generation paths have their own bug history.

## Relevant files

- [`src/nvim/spellfile.c`](../../../neovim/src/nvim/spellfile.c)

## File tree

```text
src/
  nvim/
    spellfile.c
```

## Big-picture references

- [`runtime/doc/options.txt`](../../../neovim/runtime/doc/options.txt): user-facing spell options and file configuration.
- [`runtime/doc/dev_arch.txt`](../../../neovim/runtime/doc/dev_arch.txt): the big-picture persistent-file trust boundary.
- [`runtime/doc/vim_diff.txt`](../../../neovim/runtime/doc/vim_diff.txt): useful mainly as lineage context because this subsystem is still heavily inherited.

## Recent fix / history signal

- `4f7b6083e5` / `e203257fff`: stack buffer overflows in spell file generation (#38948).
- `d31f7648ec`: Unicode character overflow fix in `mkspell` (#23760).
- Older fixes in this area often mention empty dictionaries or malformed input, which is a cue to keep hostile input central in review.

## Audit focus

- Check line tokenization assumptions in `spell_read_aff()`, especially `MAXLINELEN` and `MAXITEMCNT` handling.
- Follow affix expansion into downstream word-building helpers that write into bounded arrays or use fixed-size temporary buffers.
- Review malformed encoding/conversion paths separately from ordinary affix-rule parsing.
- Treat generation code (`:mkspell`) as equally sensitive as load-time parsing; recent bugs hit both.

## See Also

- [Swap recovery parser](16-swap-recovery.md)
- [Ex address parser](20-ex-address-parser.md)
