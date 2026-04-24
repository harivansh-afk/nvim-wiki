# Shell wildcard and backtick expansion

## Overview

This is a mixed inherited-and-Nvim-wrapped shell boundary. [`src/nvim/os/shell.c`](../../raw/neovim/src/nvim/os/shell.c) builds shell commands for wildcard expansion, while [`src/nvim/path.c`](../../raw/neovim/src/nvim/path.c) contains `expand_backtick()`, which executes backtick payloads via `get_cmd_output()`. It ranks highly because it is classic command-construction code sitting on top of user-configurable shell behavior, and it recently needed a security fix for newline injection in `glob()`.

## Relevant options and knobs

- `'shell'`: selects the shell used for command execution.
- `'shellcmdflag'`: changes how commands are handed to the shell.
- `'shellquote'` and `'shellxquote'`: affect quoting around constructed shell commands.
- `'shellredir'`: affects redirection behavior used in shell-mediated helper paths.
- `'shellslash'`: changes path rewriting semantics on Windows-like setups.

## Relevant files

- [`src/nvim/os/shell.c`](../../raw/neovim/src/nvim/os/shell.c)
- [`src/nvim/path.c`](../../raw/neovim/src/nvim/path.c)

## File tree

```text
src/
  nvim/
    os/
      shell.c
    path.c
```

## Big-picture references

- [`runtime/doc/dev_arch.txt`](../../raw/neovim/runtime/doc/dev_arch.txt): explicitly marks shell / job / PTY boundaries as trust boundaries.
- [`runtime/doc/options.txt`](../../raw/neovim/runtime/doc/options.txt): authoritative descriptions for shell-related options.
- [`runtime/doc/api.txt`](../../raw/neovim/runtime/doc/api.txt): notes that wildcard expansion and expression evaluation are not “fast” operations, which matters for reentrancy assumptions.

## Recent fix / history signal

- `f577e05522` / `bea7f3a44e`: security fix for command injection via newline in `glob()` (#38385).
- Older logic in this area already treats backticks as special and blocks them in secure contexts, which is itself a clue that the boundary is security-sensitive.

## Audit focus

- Trace which callers reach `os_expand_wildcards()` versus `expand_backtick()` and which security checks differ between them.
- Review newline, carriage-return, backtick, and shell-special escaping under all shell styles (`echo`, `glob`, `print`, `vimglob`, direct backtick execution).
- Check how temp-file based output capture interacts with command failure, partial output, and NUL-containing results.
- Treat user-configured shell options as hostile inputs for parser and escaping review, not as trusted environment.

## See Also

- [Tar plugin extraction path](01-tar-extract.md)
- [Zip plugin extraction path](02-zip-extract.md)
- [Tag jump and tag filename expansion](04-tag-jump-expand.md)
