# Trust database and :trust bridge

## Overview

This is a Neovim-specific policy surface rather than a parser or renderer. [`runtime/lua/vim/secure.lua`](../../raw/neovim/runtime/lua/vim/secure.lua) manages the trust database under `$XDG_STATE_HOME/nvim/trust`, while [`src/nvim/lua/secure.c`](../../raw/neovim/src/nvim/lua/secure.c) bridges that logic into C and `:trust`. It ranks highly because it decides whether local project code runs at all; if the policy is wrong, every higher-level exrc safeguard above it becomes moot.

## Relevant options and knobs

- `'exrc'`: the main consumer of the trust decision, even though trust itself is not a standalone option.
- `::trust` command: adds, denies, or removes entries from the trust database.
- `vim.secure.read()`: reads and prompts on untrusted files or directories.
- `vim.secure.trust()`: programmatic interface for managing trust entries.

## Relevant files

- [`runtime/lua/vim/secure.lua`](../../raw/neovim/runtime/lua/vim/secure.lua)
- [`src/nvim/lua/secure.c`](../../raw/neovim/src/nvim/lua/secure.c)

## File tree

```text
runtime/
  lua/
    vim/
      secure.lua
src/
  nvim/
    lua/
      secure.c
```

## Big-picture references

- [`runtime/doc/editing.txt`](../../raw/neovim/runtime/doc/editing.txt): the main “trusted files” and `:trust` documentation, including the TOCTOU warning.
- [`runtime/doc/lua.txt`](../../raw/neovim/runtime/doc/lua.txt): documents `vim.secure.read()` and `vim.secure.trust()`.
- [`runtime/doc/news-0.12.txt`](../../raw/neovim/runtime/doc/news-0.12.txt): captures recent trust UX and semantics changes, including directory trust and the removal of the old “allow” prompt flow.

## Recent fix / history signal

- Recent docs explicitly call out TOCTOU risk for `:trust [file]`, which is a strong sign that this policy layer has sharp edges.
- 0.12 also changed directory-trust behavior and `exrc` prompting semantics, so older assumptions are not necessarily current assumptions.

## Audit focus

- Check realpath normalization and ensure the trusted path is the same path later used for execution.
- Review file-hash versus directory-name trust semantics separately; the threat models are different.
- Audit buffer-based trust operations (`bufnr`) alongside path-based ones; they have different input sources and race characteristics.
- Treat prompt error handling and interrupt handling as security-relevant, not just UX details.

## See Also

- [Project-local exrc loader](07-exrc-loader.md)
- [Modeline execution pipeline](05-modelines.md)
- [mapset secure-mode gate](06-mapset-secure.md)
