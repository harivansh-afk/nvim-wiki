# Tag jump and tag filename expansion

## Overview

This is a mostly Vim-derived tag-file trust boundary. In [`src/nvim/tag.c`](../../raw/neovim/src/nvim/tag.c), `jumpto_tag()` and `expand_tag_fname()` turn data from tag files into filenames, search commands, and sometimes Ex command fragments. It ranks very high because recent fixes show both code-execution and memory-safety history here, and because tag files are often treated as benign text when they are really executable navigation metadata.

## Relevant options and knobs

- `'tags'`: controls which tag files are searched.
- `'tagrelative'`: rewrites relative tag filenames against the tag file location.
- `'tagfunc'`: allows user code to provide tag results and can therefore widen the input surface.
- `'helpfile'`: relevant because help tags share neighboring discovery and path logic in the same module.

## Relevant files

- [`src/nvim/tag.c`](../../raw/neovim/src/nvim/tag.c)

## File tree

```text
src/
  nvim/
    tag.c
```

## Big-picture references

- [`runtime/doc/tagsrch.txt`](../../raw/neovim/runtime/doc/tagsrch.txt): big-picture behavior for tag searches and secure restrictions around tag commands.
- [`runtime/doc/options.txt`](../../raw/neovim/runtime/doc/options.txt): option-level context for `tags`, `tagrelative`, `tagfunc`, and `helpfile`.
- [`runtime/doc/message.txt`](../../raw/neovim/runtime/doc/message.txt): relevant because some secure-mode tag failures surface as classic “command not allowed” messages.

## Recent fix / history signal

- `9c11229832` / `0e07b2a1e2`: security fix for command injection via backticks in tag files (#39102).
- `761e920280` / `3e7f0a13e1`: security overflow involving `nostartofline` and Ex commands in tag files (#32739).
- `ebfbe4db49`: crash when using `tagfunc` (#37627).

## Audit focus

- Follow the flow from raw tag line parsing to filename expansion to eventual search / command execution.
- Check how wildcard expansion in `expand_tag_fname()` composes with `tagrelative` and user-controlled tag file content.
- Review help-preview and split-window code paths for autocommand reentrancy and stale-pointer assumptions.
- Treat `tagfunc` results as equally hostile even though they come from script callbacks rather than files on disk.

## See Also

- [Tag and helpfile discovery](09-tagfile-discovery.md)
- [Shell wildcard and backtick expansion](03-shell-glob-backtick.md)
- [Ex address parser](20-ex-address-parser.md)
