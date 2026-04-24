# Persistence and parser surfaces

> Sources: [Swap recovery parser](../tasks/16-swap-recovery.md); [ShaDa item parser](../tasks/17-shada-parser.md); [Spell affix parser](../tasks/18-spell-affix-parser.md)
> Updated: 2026-04-23

## Overview

This area groups the file-driven parsers that read hostile or semi-trusted on-disk data: swap recovery metadata, MessagePack-based ShaDa state, and spell dictionaries or affix rules. These paths are not part of ordinary editing hot loops, which makes them easier to under-test and easier to misjudge as "just file I/O."

## Covered task briefs

- [Swap recovery parser](../tasks/16-swap-recovery.md)
- [ShaDa item parser](../tasks/17-shada-parser.md)
- [Spell affix parser](../tasks/18-spell-affix-parser.md)

## Key files and entrypoints

- [src/nvim/memline.c](../../raw/neovim/src/nvim/memline.c)
- [src/nvim/shada.c](../../raw/neovim/src/nvim/shada.c)
- [src/nvim/spellfile.c](../../raw/neovim/src/nvim/spellfile.c)
- Shared parser helper of interest:
  - [src/nvim/msgpack_rpc/unpacker.c](../../raw/neovim/src/nvim/msgpack_rpc/unpacker.c)

## Primary docs and knobs

- [runtime/doc/dev_arch.txt](../../raw/neovim/runtime/doc/dev_arch.txt)
- [runtime/doc/options.txt](../../raw/neovim/runtime/doc/options.txt)
- [runtime/doc/starting.txt](../../raw/neovim/runtime/doc/starting.txt)
- [runtime/doc/vim_diff.txt](../../raw/neovim/runtime/doc/vim_diff.txt)
- High-value knobs: `'directory'`, `'swapfile'`, `'shada'`, `'spellfile'`, `'spelllang'`, `:rshada`, `:wshada`, `:mkspell`

## How agents should navigate this area

1. Start by identifying the hostile file format:
   - swap recovery metadata
   - ShaDa items and compatibility records
   - spell affix and dictionary text
2. Review metadata and length fields before normal parsing loops.
3. Treat recovery, compatibility, and generation paths as equal audit targets, not side paths.
4. When parser helpers are shared across subsystems, compare the assumptions made in each caller.

## Boundary map

- `memline.c` reads recovery metadata before walking core buffer structures.
- `shada.c` decodes persisted editor state and then merges or writes it back.
- `spellfile.c` mixes parsing and generation in one subsystem.
- If the issue is really about remote MessagePack parsing rather than on-disk MessagePack, continue with [RPC and channel surfaces](rpc-and-channel-surfaces.md).

## See Also

- [RPC and channel surfaces](rpc-and-channel-surfaces.md)
- [Display and formatting surfaces](display-and-formatting-surfaces.md)
- [Task brief navigation](../agents/task-brief-navigation.md)
