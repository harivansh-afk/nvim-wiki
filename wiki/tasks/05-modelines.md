# Modeline execution pipeline

## Overview

This is one of the clearest inherited untrusted-file boundaries in the tree. [`src/nvim/buffer.c`](../../../neovim/src/nvim/buffer.c) scans file contents with `do_modelines()` and parses candidate modelines with `chk_modeline()`, while option metadata in [`src/nvim/options.lua`](../../../neovim/src/nvim/options.lua) controls which settings are blocked or restricted. It ranks near the top because plain file text can influence editor behavior, and recent history proves that even narrow policy mistakes here can become security bugs.

## Relevant options and knobs

- `'modeline'`: master on/off switch for modeline processing.
- `'modelines'`: controls how many lines from the start and end of the file are scanned.
- `'modelineexpr'`: decides whether expression-capable options may be set from modelines.
- Options marked `secure = true` in `options.lua`: these are the generated denylist / policy layer for modelines and sandbox contexts.

## Relevant files

- [`src/nvim/buffer.c`](../../../neovim/src/nvim/buffer.c)
- [`src/nvim/options.lua`](../../../neovim/src/nvim/options.lua)

## File tree

```text
src/
  nvim/
    buffer.c
    options.lua
```

## Big-picture references

- [`runtime/doc/options.txt`](../../../neovim/runtime/doc/options.txt): the main user-facing documentation for `modeline`, `modelines`, and `modelineexpr`.
- [`runtime/doc/starting.txt`](../../../neovim/runtime/doc/starting.txt): documents the long-standing warnings about unsafe local configuration and untrusted files.
- [`runtime/doc/vim_diff.txt`](../../../neovim/runtime/doc/vim_diff.txt): useful for separating legacy Vim semantics from Nvim trust-model changes.

## Recent fix / history signal

- `c7604323e3` / `c084ab9f57`: security fix for a modeline security bypass (#38657).
- The bypass touched both runtime option metadata and individual execution gates, which is a strong signal that this pipeline is easy to get subtly wrong.

## Audit focus

- Review recursion and autocommand re-entry: `do_modelines()` already has a static recursion guard for a reason.
- Check the exact parsing of `vi:`, `vim:`, version-gated prefixes, escaped colons, and option separators.
- Diff the generated option policy in `options.lua` against all code paths that actually apply options from modelines.
- Treat “expression-capable” options as higher risk than plain scalar options, even if they are only indirectly evaluated later.

## See Also

- [mapset secure-mode gate](06-mapset-secure.md)
- [Project-local exrc loader](07-exrc-loader.md)
- [Trust database and :trust bridge](08-trust-db.md)
