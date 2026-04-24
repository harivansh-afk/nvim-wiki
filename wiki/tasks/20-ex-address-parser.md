# Ex address parser

## Overview

This is a mostly Vim-derived command parser boundary. [`src/nvim/ex_docmd.c`](../../raw/neovim/src/nvim/ex_docmd.c) implements `parse_cmd_address()` and `get_address()`, which walk raw Ex command text to interpret ranges, marks, searches, offsets, and special forms like `%` or `*`. It ranks highly because this is hand-rolled pointer-walking parser code with explicit security overflow history, and it sits in front of a huge amount of command execution logic.

## Relevant options and knobs

- There is no single editor option for this parser; the surface is exposed by Ex commands that accept ranges.
- Operational knobs include the current buffer/window state, mark tables, and search pattern semantics rather than user-set options.
- Related command behavior and compatibility flags can still affect downstream assumptions, so treat context as part of the input.

## Relevant files

- [`src/nvim/ex_docmd.c`](../../raw/neovim/src/nvim/ex_docmd.c)

## File tree

```text
src/
  nvim/
    ex_docmd.c
```

## Big-picture references

- [`runtime/doc/dev_arch.txt`](../../raw/neovim/runtime/doc/dev_arch.txt): places Ex/Vimscript command execution in the core editor subsystem.
- [`runtime/doc/vim_diff.txt`](../../raw/neovim/runtime/doc/vim_diff.txt): useful for command-behavior differences that can change expectations around range parsing.
- [`runtime/doc/message.txt`](../../raw/neovim/runtime/doc/message.txt): relevant for the concrete error surfaces that parser failures expose.

## Recent fix / history signal

- `809b05bf27`: security overflow in Ex address parsing.
- Nearby history also includes large-count and range-related overflow fixes, which suggests the parser remains fragile under unusual numeric input.

## Audit focus

- Review raw pointer walking and the way `parse_cmd_address()` mutates command pointers and sometimes cursor state before final success/failure is known.
- Check arithmetic on large counts, offsets, and derived line numbers under both normal and error flows.
- Treat search delimiters, other-file marks, visual-range forms, and `;`-separated cursor-moving ranges as separate subparsers.
- Exercise inputs that are syntactically odd but still mostly valid; parser bugs often hide between clean success and clean failure cases.

## See Also

- [Tag jump and tag filename expansion](04-tag-jump-expand.md)
- [Statusline mini-language parser](19-statusline-parser.md)
