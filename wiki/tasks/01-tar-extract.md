# Tar plugin extraction path

## Overview

This is a mostly Vim-derived runtime-plugin boundary. `tar#Extract()` in [`runtime/autoload/tar.vim`](../../../neovim/runtime/autoload/tar.vim) turns the archive member name under the cursor into a shell-facing extraction command. That makes it high risk because attacker-controlled tar metadata can cross from archive parsing into process execution and filesystem writes. It ranks near the top because recent history already includes crafted-archive code execution, path-traversal fixes, and hardening around command construction.

## Relevant options and knobs

- `g:tar_extractcmd`: selects the extraction command string used by `tar#Extract()`.
- `g:tar_cmd`: affects the broader tar plugin command path and shell command assembly.
- `g:tar_secure`: separator inserted before extracted member names; recent fixes show this knob matters.
- `&shell`: indirectly matters because `system()` semantics and quoting behavior depend on the configured shell.

## Relevant files

- [`runtime/autoload/tar.vim`](../../../neovim/runtime/autoload/tar.vim)

## File tree

```text
runtime/
  autoload/
    tar.vim
```

## Big-picture references

- [`runtime/doc/dev_arch.txt`](../../../neovim/runtime/doc/dev_arch.txt): frames runtime plugins as a major user-facing surface under `runtime/`.
- [`runtime/doc/vim_diff.txt`](../../../neovim/runtime/doc/vim_diff.txt): useful for judging how much runtime behavior is inherited from Vim versus Nvim-specific.
- [`src/nvim/options.lua`](../../../neovim/src/nvim/options.lua): relevant indirectly for shell-related option metadata that influences command execution semantics.

## Recent fix / history signal

- `560b8a8ce0`: security fix for code execution with specially crafted tar files (#32701).
- `77c6cae25b`: security fix for a tar.vim path traversal issue.
- `c3c06723f0`: missing path traversal checks in `tar#Extract()` (#39095).
- `3cca237984`: missing `g:tar_secure` in `tar#Extract()` (#39123).

## Audit focus

- Follow the exact data flow from the archive member name (`getline(".")`) into `system()` arguments.
- Check traversal normalization across plain tar, gzip, bzip2, zstd, lz4, and platform-specific extract command rewrites.
- Review whether command-string assembly stays safe when users override `g:tar_extractcmd` or related globals.
- Pay special attention to quoting boundaries that look safe in one shell but not another.

## See Also

- [Zip plugin extraction path](02-zip-extract.md)
- [Shell wildcard and backtick expansion](03-shell-glob-backtick.md)
