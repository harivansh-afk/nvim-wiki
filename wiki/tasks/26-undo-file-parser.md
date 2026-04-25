# Undo file parser

## Overview

Persisted-undo is a hostile-file parser surface analogous to swap/ShaDa: when Neovim opens a buffer whose `'undofile'` exists on disk, [`src/nvim/undo.c`](../../raw/neovim/src/nvim/undo.c)'s `u_read_undo()` parses an attacker-influenceable binary file. With `'undofile'` on (default off but commonly enabled), opening a tracked file triggers parser execution before any user interaction. Listed as a high-yield missing surface in [`runtime.md` rule 44](runtime.md).

## Relevant options and knobs

- `'undofile'`: master on/off (default off, but very commonly enabled in dotfiles).
- `'undodir'`: where undo files are searched. Attacker-writable `'undodir'` entries are a planting vector.
- `:rundo`, `:wundo`: explicit operational entrypoints.

## Relevant files

- [`src/nvim/undo.c`](../../raw/neovim/src/nvim/undo.c) (`u_read_undo`, `u_write_undo`, magic-number/version checks)
- [`src/nvim/sha256.c`](../../raw/neovim/src/nvim/sha256.c) (used as a content-integrity check)

## File tree

```text
src/nvim/
  undo.c
  sha256.c
```

## Big-picture references

- [`runtime/doc/undo.txt`](../../raw/neovim/runtime/doc/undo.txt): undo-file format and trust posture.
- [`runtime/doc/options.txt`](../../raw/neovim/runtime/doc/options.txt) `*'undofile'*` `*'undodir'*`.

## Recent fix / history signal

- Vim CVE history around `u_read_undo` (length fields, header parsing): variant-hunt these in Nvim — most weren't ported because Nvim refactored the parser, but precondition drift may still leave reachable cases.
- See [`runtime.md`](runtime.md) rule 44 for the maintainer-signal that this surface is under-audited.

## Audit focus

- Map every length-prefixed field in `u_read_undo` and prove the corresponding allocation/check matches. Classic OOB shape: header says N, file has < N, parser reads past EOF.
- Magic / version / hash mismatch paths: do they fail safely or do they leave partial state in the buffer?
- Concurrent-edit handling: opening the same file in two Nvim instances — does undo replay race against fresh writes?
- Cross-check `'undodir'` resolution for the same symlink-following pattern that landed [GHSA-3r8v](../cves/GHSA-3r8v.md).

## See Also

- [Swap recovery parser](16-swap-recovery.md)
- [ShaDa item parser](17-shada-parser.md)
- [GHSA-3r8v (file race)](../cves/GHSA-3r8v.md)
