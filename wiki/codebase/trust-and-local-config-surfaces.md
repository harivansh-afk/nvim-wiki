# Trust and local config surfaces

> Sources: [Modeline execution pipeline](../tasks/05-modelines.md); [mapset secure-mode gate](../tasks/06-mapset-secure.md); [Project-local exrc loader](../tasks/07-exrc-loader.md); [Trust database and :trust bridge](../tasks/08-trust-db.md)
> Updated: 2026-04-23

## Overview

This area is the trust-policy spine for local configuration and file-driven behavior. It connects modelines, generated option policy, secure-mode gates, project-local config discovery, and the trust database that ultimately decides whether a local file is allowed to execute.

## Covered task briefs

- [Modeline execution pipeline](../tasks/05-modelines.md)
- [mapset secure-mode gate](../tasks/06-mapset-secure.md)
- [Project-local exrc loader](../tasks/07-exrc-loader.md)
- [Trust database and :trust bridge](../tasks/08-trust-db.md)

## Key files and entrypoints

- Modeline and option policy:
  - [src/nvim/buffer.c](../../raw/neovim/src/nvim/buffer.c)
  - [src/nvim/options.lua](../../raw/neovim/src/nvim/options.lua)
- Secure-mode gate:
  - [src/nvim/mapping.c](../../raw/neovim/src/nvim/mapping.c)
  - [src/nvim/ex_cmds.c](../../raw/neovim/src/nvim/ex_cmds.c)
- Project-local config loading:
  - [src/nvim/main.c](../../raw/neovim/src/nvim/main.c)
  - [src/nvim/runtime.c](../../raw/neovim/src/nvim/runtime.c)
  - [runtime/lua/vim/_core/exrc.lua](../../raw/neovim/runtime/lua/vim/_core/exrc.lua)
- Trust database:
  - [runtime/lua/vim/secure.lua](../../raw/neovim/runtime/lua/vim/secure.lua)
  - [src/nvim/lua/secure.c](../../raw/neovim/src/nvim/lua/secure.c)

## Primary docs and knobs

- [runtime/doc/options.txt](../../raw/neovim/runtime/doc/options.txt)
- [runtime/doc/starting.txt](../../raw/neovim/runtime/doc/starting.txt)
- [runtime/doc/editing.txt](../../raw/neovim/runtime/doc/editing.txt)
- [runtime/doc/lua.txt](../../raw/neovim/runtime/doc/lua.txt)
- [runtime/doc/news-0.12.txt](../../raw/neovim/runtime/doc/news-0.12.txt)
- High-value knobs: `'modeline'`, `'modelines'`, `'modelineexpr'`, `'exrc'`, `:trust`, `vim.secure.read()`, `vim.secure.trust()`

## How agents should navigate this area

1. Start from the earliest trust decision in your flow:
   - file contents -> modelines
   - restricted context -> `check_secure()`
   - project startup -> `exrc`
   - path approval -> trust database
2. Compare generated policy in `options.lua` against handwritten C or Lua checks.
3. Treat normalized paths, execution context, and reentrancy as first-class concerns, not edge cases.
4. If a task crosses into tags or command text, continue with [Tag and command surfaces](tag-and-command-surfaces.md).

## Boundary map

- Modelines decide what file text may configure.
- `check_secure()` decides what restricted contexts may execute.
- `exrc` decides which project-local files are candidates for execution.
- The trust database decides whether those candidates are allowed to run at all.

## See Also

- [Tag and command surfaces](tag-and-command-surfaces.md)
- [Task brief navigation](../agents/task-brief-navigation.md)
- [Neovim contribution and maintenance workflow](../neovim/contribution-and-maintenance-workflow.md)
