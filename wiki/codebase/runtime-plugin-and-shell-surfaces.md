# Runtime plugin and shell surfaces

> Sources: [Tar plugin extraction path](../tasks/01-tar-extract.md); [Zip plugin extraction path](../tasks/02-zip-extract.md); [Shell wildcard and backtick expansion](../tasks/03-shell-glob-backtick.md); [Neovim project goals and layout](../neovim/project-goals-and-layout.md)
> Updated: 2026-04-23

## Overview

This area covers the command-construction surfaces where filenames, archive member names, wildcard expressions, or backtick payloads can turn into shell execution or filesystem writes. Agents working here should treat the runtime plugin layer and the core shell/path layer as connected, because `tar.vim` and `zip.vim` both depend on shell semantics that are also embodied in `shell.c` and `path.c`.

## Covered task briefs

- [Tar plugin extraction path](../tasks/01-tar-extract.md)
- [Zip plugin extraction path](../tasks/02-zip-extract.md)
- [Shell wildcard and backtick expansion](../tasks/03-shell-glob-backtick.md)

## Key files and entrypoints

- Runtime plugin extraction:
  - [runtime/autoload/tar.vim](../../raw/neovim/runtime/autoload/tar.vim)
  - [runtime/autoload/zip.vim](../../raw/neovim/runtime/autoload/zip.vim)
  - [runtime/doc/pi_zip.txt](../../raw/neovim/runtime/doc/pi_zip.txt)
- Core shell and path execution:
  - [src/nvim/os/shell.c](../../raw/neovim/src/nvim/os/shell.c)
  - [src/nvim/path.c](../../raw/neovim/src/nvim/path.c)

## Primary docs and knobs

- [runtime/doc/dev_arch.txt](../../raw/neovim/runtime/doc/dev_arch.txt)
- [runtime/doc/options.txt](../../raw/neovim/runtime/doc/options.txt)
- [runtime/doc/vim_diff.txt](../../raw/neovim/runtime/doc/vim_diff.txt)
- High-value knobs: `g:tar_extractcmd`, `g:tar_cmd`, `g:tar_secure`, `g:zip_extractcmd`, `g:zip_unzipcmd`, `'shell'`, `'shellcmdflag'`, `'shellquote'`, `'shellxquote'`, `'shellredir'`, `'shellslash'`

## How agents should navigate this area

1. Start with the exact task brief for the surface you own.
2. Follow the input boundary that turns parsed text into command strings:
   - archive member name -> extraction command
   - wildcard or backtick syntax -> shell execution or output capture
3. Compare platform or shell-specific branches separately. GNU shell, PowerShell, and user-overridden shell settings do not share identical escaping rules.
4. Use the core shell/path code as the shared lower layer whenever a plugin path looks safe only because of quoting assumptions.

## Boundary map

- `tar.vim` and `zip.vim` are user-facing runtime plugins under `runtime/`.
- `shell.c` and `path.c` are core helpers that influence or mirror the same command-construction risks.
- If the issue shifts from shell construction into tag expansion or Ex command parsing, continue with [Tag and command surfaces](tag-and-command-surfaces.md).

## See Also

- [Task brief navigation](../agents/task-brief-navigation.md)
- [Tag and command surfaces](tag-and-command-surfaces.md)
- [Neovim project goals and layout](../neovim/project-goals-and-layout.md)
