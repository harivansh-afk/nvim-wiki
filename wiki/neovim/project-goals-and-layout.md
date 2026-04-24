# Neovim project goals and layout

> Sources: Neovim project docs, Unknown
> Raw: [README](../../raw/neovim/README.md); [Developer architecture reference](../../raw/neovim/runtime/doc/dev_arch.txt); [Developer tools quickstart](../../raw/neovim/runtime/doc/dev_tools.txt)
> Updated: 2026-04-23

## Overview

Neovim presents itself as an aggressive refactoring of Vim that should be easier to maintain, easier to contribute to, and more open to external UIs and API-driven tooling. The repository layout reflects that goal by separating core C code, runtime Lua, generated metadata, UI and RPC interfaces, and test infrastructure.

## Project goals

- Simplify maintenance and encourage contributions.
- Split work across multiple developers instead of concentrating knowledge in one place.
- Enable advanced UIs without requiring core changes for each UI.
- Maximize extensibility through API access, RPC, and runtime Lua modules while remaining compatible with most Vim plugins.

## Repository layout

- `src/nvim/` contains the main editor implementation in C, including API, eval, event loop, Lua, RPC, OS, and TUI subsystems.
- `src/*` also contains vendored or owned third-party components. Some are fully owned forks, while others are synced from upstream and should not be treated as local redesigns.
- `runtime/` contains user-facing runtime assets such as docs, plugins, and Lua modules.
- `test/` is divided into benchmark, functional, unit, and legacy test areas.
- `cmake/`, `cmake.config/`, and `cmake.deps/` hold build configuration and dependency-fetching logic.

## Architecture and extension model

Lua code is intentionally split by role:

- User-facing or replaceable features belong in plugins under `runtime/plugin/` or `runtime/pack/dist/opt/`.
- Lazy-loaded standard library modules live under `runtime/lua/vim/`.
- Internal compiled-in modules that must work even when `VIMRUNTIME` is unavailable live under `runtime/lua/vim/_core/`.
- Shared pure-Lua helpers that need to work in tests and worker contexts are kept in `runtime/lua/vim/_core/shared.lua`.

The developer architecture docs also describe a data-driven design where many editor concepts are declared in Lua metadata files, including events, Ex commands, options, Vimscript functions, and `v:` variables. This keeps important editor surfaces centralized and generates parts of the implementation and documentation from those definitions.

## UI, API, and event boundaries

- High-level editor events are intended for plugins, configuration, and integrations.
- UI events currently travel through `redraw` notifications over RPC and can also be observed in-process through `vim.ui_attach()`.
- New functionality is generally preferred in Lua instead of C when feasible, because that keeps more behavior in higher-level code and aligns with the project’s maintenance goals.

## See Also

- [Neovim build, install, and test workflow](build-install-and-test-workflow.md)
- [Neovim contribution and maintenance workflow](contribution-and-maintenance-workflow.md)
