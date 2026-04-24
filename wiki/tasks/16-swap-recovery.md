# Swap recovery parser

## Overview

This is a mostly Vim-derived hostile-file parser. [`src/nvim/memline.c`](../../raw/neovim/src/nvim/memline.c) uses `ml_recover()` to read untrusted swap metadata such as block numbers, page counts, and line-count hints before walking the memline tree. It ranks highly because malformed swapfiles previously triggered crashes and potentially huge allocations, and because recovery code is easy to under-test relative to normal editing paths.

## Relevant options and knobs

- `'directory'`: controls where swap files are created and therefore where recovery input comes from.
- `'swapfile'`: master switch for swap-file creation.
- `-r` / recovery mode: operationally relevant even though it is startup behavior rather than an option.

## Relevant files

- [`src/nvim/memline.c`](../../raw/neovim/src/nvim/memline.c)

## File tree

```text
src/
  nvim/
    memline.c
```

## Big-picture references

- [`runtime/doc/dev_arch.txt`](../../raw/neovim/runtime/doc/dev_arch.txt): explains the memline tree as the core buffer-text data structure.
- [`runtime/doc/options.txt`](../../raw/neovim/runtime/doc/options.txt): documents `directory` and `swapfile` behavior.
- [`runtime/doc/starting.txt`](../../raw/neovim/runtime/doc/starting.txt): useful for recovery-mode and startup-file interaction context.

## Recent fix / history signal

- `7e8bdd348c` / `7fc228d94f`: security fix for crash when recovering a corrupted swap file (#38104).
- The hardening explicitly validated `pe_line_count`, `pe_old_lnum`, `pe_bnum`, and `pe_page_count`, which identifies the exact risky fields to revisit.

## Audit focus

- Re-review every place where untrusted swap metadata is used before a bounds or range check.
- Check integer overflow on block arithmetic, especially `bnum + page_count` style computations.
- Treat “keep recovering whatever else we can” error paths as security-relevant because partial recovery still walks hostile structures.
- Exercise impossible-but-parseable metadata combinations, not just obviously corrupt files.

## See Also

- [ShaDa item parser](17-shada-parser.md)
- [Spell affix parser](18-spell-affix-parser.md)
