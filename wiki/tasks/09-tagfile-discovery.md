# Tag and helpfile discovery

## Overview

This is a mostly Vim-derived path-construction region adjacent to the tag jump pipeline. [`src/nvim/tag.c`](../../../neovim/src/nvim/tag.c) uses `get_tagfname()` to discover tag files, with special handling for help tags and a fallback that rewrites `helpfile` into a `tags` path. It earns its own ticket because recent security history includes explicit `helpfile` overflows, which means the path-discovery side of `tag.c` is not merely administrative glue.

## Relevant options and knobs

- `'tags'`: controls normal tag-file search paths.
- `'helpfile'`: participates in the help-mode fallback path and has direct bug history here.
- `'runtimepath'`: determines how help tags are discovered across installed runtime trees.

## Relevant files

- [`src/nvim/tag.c`](../../../neovim/src/nvim/tag.c)

## File tree

```text
src/
  nvim/
    tag.c
```

## Big-picture references

- [`runtime/doc/tagsrch.txt`](../../../neovim/runtime/doc/tagsrch.txt): user-facing tag search behavior.
- [`runtime/doc/options.txt`](../../../neovim/runtime/doc/options.txt): documents `tags`, `helpfile`, and `runtimepath`-adjacent behavior.
- [`runtime/doc/dev_arch.txt`](../../../neovim/runtime/doc/dev_arch.txt): describes runtime path loading, useful for understanding help-tag discovery context.

## Recent fix / history signal

- `db133879b2` / `4792c29969`: security fix for buffer overflow in `helpfile` option handling (#37735).
- `15061d322d` / `03e68ad5d3`: follow-up fix for another helpfile overflow case (#37746).

## Audit focus

- Review every MAXPATHL-sized copy or truncation in the help-mode branch of `get_tagfname()`.
- Check how `runtimepath`-derived help tag results interact with the `helpfile` fallback and duplicate suppression.
- Treat “path building for docs” as security-relevant because the historical bug class here was memory safety, not just wrong-path behavior.
- Compare normal tag lookup and help-tag lookup separately; the control flow and assumptions differ meaningfully.

## See Also

- [Tag jump and tag filename expansion](04-tag-jump-expand.md)
- [Modeline execution pipeline](05-modelines.md)
