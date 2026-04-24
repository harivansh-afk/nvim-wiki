# `:write` backup-copy filesystem race

## Overview

This is the surface that produced [GHSA-3r8v](../cves/GHSA-3r8v.md) — *"Symlink race in `:write` backup-copy path can overwrite arbitrary files"*, found by Devin AI under supervision and accepted at CVSS 6.1. The `os_remove` → `os_copy` window in [`buf_write_make_backup`](../../raw/neovim/src/nvim/bufwrite.c) is the canonical case, but the *class* (TOCTOU on file write paths without `O_NOFOLLOW`/`O_EXCL`/fd-pinning) is broader: backup, swap, undo, ShaDa, and `:trust` db all do "remove-then-recreate" or "rename-then-fsync" without race-safe primitives.

## Relevant options and knobs

- `'writebackup'`, `'backup'`, `'backupdir'`, `'backupcopy'`: trigger surface for the GHSA-3r8v case. Defaults make link-backed files reach the copy path.
- `'directory'` (swap), `'undodir'` (undo), `vim.fn.stdpath('state')` (trust db): sibling write surfaces.
- File-mode and ownership behavior — preservation requirements drive the unsafe rename/copy patterns.

## Relevant files

- [`src/nvim/bufwrite.c`](../../raw/neovim/src/nvim/bufwrite.c) (`buf_write_make_backup`, `buf_write`)
- [`src/nvim/os/fs.c`](../../raw/neovim/src/nvim/os/fs.c) (`os_remove`, `os_copy`, `os_rename`)
- [`runtime/lua/vim/secure.lua`](../../raw/neovim/runtime/lua/vim/secure.lua) (trust-db write path)
- [`src/nvim/memline.c`](../../raw/neovim/src/nvim/memline.c) (swap write paths)

## File tree

```text
src/nvim/
  bufwrite.c
  memline.c
  os/fs.c
runtime/lua/vim/
  secure.lua
```

## Big-picture references

- [GHSA-3r8v advisory](https://github.com/neovim/neovim/security/advisories/GHSA-3r8v-crjq-wphr)
- [`runtime/doc/editing.txt`](../../raw/neovim/runtime/doc/editing.txt): `:trust [file]` documents a TOCTOU risk explicitly — quoting from there in your finding is high-leverage.

## Recent fix / history signal

- [GHSA-3r8v](../cves/GHSA-3r8v.md) — the canonical example. The fix landed (or not) in `bufwrite.c`; sibling write paths likely still have the same shape.

## Audit focus

- Variant-hunt the GHSA-3r8v shape. Every `os_remove(target); os_copy(src, target)` and `os_rename(tmp, target)` pair is a candidate. List them all, then prove which can be raced by an attacker with write access to the parent dir.
- `:trust` write path in `secure.lua` — the doc admits TOCTOU on `:trust [file]`. A finding here would be Tier-A if it overwrites arbitrary files at the user's privilege.
- Swap/undo/ShaDa: `mch_open_rw` and equivalents — verify `O_EXCL`/`O_NOFOLLOW` on the create path.
- Windows code branches: separate primitives, often missed in audits.

## See Also

- [GHSA-3r8v (accepted)](../cves/GHSA-3r8v.md)
- [Trust database and :trust bridge](08-trust-db.md)
- [Swap recovery parser](16-swap-recovery.md)
