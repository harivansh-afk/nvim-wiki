# Tag and command surfaces

> Sources: [Tag jump and tag filename expansion](../tasks/04-tag-jump-expand.md); [Tag and helpfile discovery](../tasks/09-tagfile-discovery.md); [Ex address parser](../tasks/20-ex-address-parser.md)
> Updated: 2026-04-23

## Overview

This area covers the command-facing text parsers and path builders around tag navigation and Ex range handling. Agents should treat `tag.c` and `ex_docmd.c` as neighboring surfaces: tag files can feed filenames and search commands into the editor, while Ex address parsing sits directly in front of a large amount of later command execution logic.

## Covered task briefs

- [Tag jump and tag filename expansion](../tasks/04-tag-jump-expand.md)
- [Tag and helpfile discovery](../tasks/09-tagfile-discovery.md)
- [Ex address parser](../tasks/20-ex-address-parser.md)

## Key files and entrypoints

- [src/nvim/tag.c](../../raw/neovim/src/nvim/tag.c)
- [src/nvim/ex_docmd.c](../../raw/neovim/src/nvim/ex_docmd.c)
- Supporting docs:
  - [runtime/doc/tagsrch.txt](../../raw/neovim/runtime/doc/tagsrch.txt)
  - [runtime/doc/options.txt](../../raw/neovim/runtime/doc/options.txt)
  - [runtime/doc/message.txt](../../raw/neovim/runtime/doc/message.txt)

## Primary docs and knobs

- Tag-related knobs: `'tags'`, `'tagrelative'`, `'tagfunc'`, `'helpfile'`, `'runtimepath'`
- Ex address parsing is shaped more by command syntax and editor state than by a single option, so marks, search delimiters, offsets, and current buffer/window state matter as inputs.

## How agents should navigate this area

1. Decide whether your input enters through tag-file discovery, tag-line execution, or raw Ex command text.
2. For tag issues, separate discovery logic from jump/execution logic:
   - `get_tagfname()` and help-tag discovery
   - `jumpto_tag()` and `expand_tag_fname()`
3. For Ex issues, isolate the exact subparser first: marks, searches, `%`, `*`, offsets, or `;`-driven cursor moves.
4. If shell expansion becomes part of the flow, branch into [Runtime plugin and shell surfaces](runtime-plugin-and-shell-surfaces.md).

## Boundary map

- `tag.c` mixes path building, tag-file discovery, help-tag discovery, and tag-jump execution.
- `ex_docmd.c` is a front-door parser for command ranges, so malformed command text can reach deep execution paths.
- Both areas have security history that mixes memory safety with command-text trust problems.

## See Also

- [Runtime plugin and shell surfaces](runtime-plugin-and-shell-surfaces.md)
- [Trust and local config surfaces](trust-and-local-config-surfaces.md)
- [Task brief navigation](../agents/task-brief-navigation.md)
