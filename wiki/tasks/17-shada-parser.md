# ShaDa item parser

## Overview

This is a mixed inherited-concept but Nvim-specific binary-format parser. [`src/nvim/shada.c`](../../../neovim/src/nvim/shada.c) uses `shada_read_next_item()` to decode persisted editor state from a MessagePack-based file format. It ranks highly because it processes attacker-controlled on-disk data, supports compatibility and unknown-item handling, and already has real out-of-bounds and use-after-free history.

## Relevant options and knobs

- `'shada'`: main option governing what state is persisted and read back.
- `::rshada` and `:wshada`: direct operational entrypoints into the persistence layer.
- Compatibility and size-limiting flags act like internal knobs even though they are not user-visible options.

## Relevant files

- [`src/nvim/shada.c`](../../../neovim/src/nvim/shada.c)
- [`src/nvim/msgpack_rpc/unpacker.c`](../../../neovim/src/nvim/msgpack_rpc/unpacker.c)

## File tree

```text
src/
  nvim/
    shada.c
    msgpack_rpc/
      unpacker.c
```

## Big-picture references

- [`runtime/doc/vim_diff.txt`](../../../neovim/runtime/doc/vim_diff.txt): explains the move from Viminfo text files to binary MessagePack ShaDa.
- [`runtime/doc/options.txt`](../../../neovim/runtime/doc/options.txt): documents the `shada` option and related persistence behavior.
- [`runtime/doc/dev_arch.txt`](../../../neovim/runtime/doc/dev_arch.txt): identifies file I/O and persisted editor state as a trust boundary.

## Recent fix / history signal

- `6bee2f686f`: use-after-free when mapping file marks (#34446).
- `ff7832ad3f`: out-of-bounds read in `shada_read_when_writing` (#30665).
- `a2008118a0`: index out-of-bounds in `shada_read_when_writing` (#30686).

## Audit focus

- Review item-length handling before buffer allocation or buffered reads.
- Treat “verify but ignore” and “unknown item” compatibility paths as first-class parser logic, not harmless extras.
- Check merge/writeback paths separately from plain read paths; several historical bugs lived there rather than in simple startup reads.
- Revisit any shared msgpack helper reused outside RPC, because parser assumptions may differ between transport and file contexts.

## See Also

- [Swap recovery parser](16-swap-recovery.md)
- [MessagePack-RPC unpacker](10-rpc-unpacker.md)
