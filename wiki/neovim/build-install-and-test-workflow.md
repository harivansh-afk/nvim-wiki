# Neovim build, install, and test workflow

> Sources: Neovim project docs, Unknown
> Raw: [INSTALL](../../raw/neovim/INSTALL.md); [BUILD](../../raw/neovim/BUILD.md); [Developer tools quickstart](../../raw/neovim/runtime/doc/dev_tools.txt); [Developer test reference](../../raw/neovim/runtime/doc/dev_test.txt)
> Updated: 2026-04-23

## Overview

Neovim supports two main paths: consume prebuilt releases through package managers or downloaded artifacts, or build the editor from source for development and unsupported environments. The repository documentation also treats testing as a first-class part of the workflow, with separate commands and harnesses for unit, functional, benchmark, and legacy test suites.

## Installation paths

- End users are expected to start with release artifacts or distribution packages on Windows, macOS, Linux, BSD, and Android.
- The executable is invoked as `nvim`, not `neovim`.
- Before upgrading, the docs recommend checking the breaking-changes section of the release notes.
- Building from source is the fallback when packages are unavailable, when a custom install prefix is needed, or when working on Neovim itself.
- Provider integrations may need extra language-specific modules. For Python plugins, the install guide recommends installing `pynvim`.

## Build workflow

- The core build system is CMake-based, with `make` provided as a convenience wrapper.
- A typical local source build from the repo root uses `make CMAKE_BUILD_TYPE=RelWithDebInfo`, then optionally `make install`.
- Output binaries land in `build/bin`, and developers can run an uninstalled editor with `VIMRUNTIME=runtime ./build/bin/nvim`.
- Common build types are `Release`, `Debug`, and `RelWithDebInfo`, depending on whether the priority is packaged output, debugger visibility, or a balance of both.
- Third-party dependencies are fetched into `.deps/` by default. `ninja` is preferred when available, and `ccache` is supported for faster rebuilds.
- When changing cached CMake values such as build type or install prefix, or after file layout changes, the docs recommend clearing `build/` or using `make distclean`.
- Windows builds are supported through several routes, but MSVC is the recommended path.

## Test workflow

- Tests are divided into unit tests, functional tests, benchmarks, and legacy Vim-derived tests.
- Unit tests compile into a shared library that is exercised through LuaJIT FFI.
- Functional tests drive a real Nvim instance over RPC and serve as fast whole-system tests.
- Common entry points are `make test`, `make unittest`, `make functionaltest`, and `make oldtest`.
- The Lua test harness loads helper modules, restores a baseline between spec files, and works well with test cases that call `clear()` to start a fresh Nvim child process.

## Debugging and verification

- For failing tests or low-level issues, the main logs are written to `$NVIM_LOG_FILE` or `build/nvim.log`.
- UI-oriented tests commonly assert through `screen:expect()`, and the screen helper can snapshot state for debugging.
- The docs also describe debugger-oriented workflows using gdbserver, Valgrind, ASan, UBSan, and TUI-specific debugging techniques.

## See Also

- [Neovim project goals and layout](project-goals-and-layout.md)
- [Neovim contribution and maintenance workflow](contribution-and-maintenance-workflow.md)
