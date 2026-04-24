# Autocmd deferred execution and reentrancy

## Overview

Autocmds run user/plugin code at well-known editor events. Many bugs in Vim/Nvim have lived in the *ordering* — what state is mutated mid-callback, what fires recursively, what gets deferred to the event loop. `runtime.md` rules 25–26 call this out: *"Reentrancy bugs matter more in Neovim than in many C projects because autocommands, Lua, and RPC can run user code almost anywhere. Any function that fires callbacks before pinning object lifetimes is a prime UAF candidate."* This task is the variant-hunt surface for that class.

## Relevant options and knobs

- `'eventignore'`: suppresses events — bypasses can produce TOCTOU between gate and use.
- `:doautocmd`, `nvim_exec_autocmds`: explicit fire points.
- `vim.schedule`, `vim.defer_fn`, `vim.uv` callbacks: deferred re-entry.

## Relevant files

- [`src/nvim/autocmd.c`](../../raw/neovim/src/nvim/autocmd.c) (`apply_autocmds_group`, `block_autocmds`, recursion guards)
- [`src/nvim/buffer.c`](../../raw/neovim/src/nvim/buffer.c) (autocmd fires during buffer lifecycle)
- [`src/nvim/lua/executor.c`](../../raw/neovim/src/nvim/lua/executor.c) (Lua callback invocation)

## File tree

```text
src/nvim/
  autocmd.c
  buffer.c
  lua/executor.c
```

## Big-picture references

- [`runtime/doc/autocmd.txt`](../../raw/neovim/runtime/doc/autocmd.txt): event list and ordering guarantees.
- [`runtime.md`](runtime.md) rules 25–26, 33–34 (Nvim async / lifetime hazards).

## Recent fix / history signal

- Variant-hunt: every UAF/double-free in `buffer.c`/`window.c` historically tied to autocmds firing mid-operation. The two retracted Vim CVEs ([`CVE-2024-41965`](../cves/CVE-2024-41965.md), [`CVE-2025-26603`](../cves/CVE-2025-26603.md)) both lived adjacent to this surface.
- [GHSA-3r8v](../cves/GHSA-3r8v.md): the `:write` race involves `BufWritePre`/`BufWritePost` ordering — adjacent.

## Audit focus

- Find functions that take a buffer/window pointer, fire an autocmd, then deref the pointer. Classic UAF.
- `block_autocmds`/`unblock_autocmds` pairs: mismatched bracketing leaves the gate stuck open or closed.
- `'eventignore'` set from a modeline-reachable code path? That's a policy gate the agent should check.
- `vim.schedule` from Lua autocmds: does the deferred work see fresh state or stale captures?
- Cross-cutting: autocmds during recovery/swap/ShaDa replay (links to `16`/`17`).

## See Also

- [Swap recovery parser](16-swap-recovery.md)
- [ShaDa item parser](17-shada-parser.md)
- [`runtime.md`](runtime.md) (rules 25–26)
