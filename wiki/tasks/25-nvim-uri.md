# `nvim://` URI scheme

## Overview

New surface added in [#38006](https://github.com/neovim/neovim/pull/38006) — registers `nvim://` as an OS-level URI handler so external tools can open files in a running Neovim. The maintainer threat model lists this as in-scope ([#38985 §1.6](https://github.com/neovim/neovim/issues/38985)). The natural attack is *click a `nvim://` link in a browser/email → URL parameters reach a sink without consent*. Classic web-style URI parsing pitfalls (path traversal, query-string injection, scheme confusion) apply.

## Relevant options and knobs

- The OS-registered URI handler — Neovim binary invoked with the URI as an argument.
- `--remote`, `--remote-send`, `--server` flags reachable via the URI parser.
- `'mouse'`, `gx`, and the `vim.ui.open` chain (where users click links inside Neovim itself).

## Relevant files

- [PR #38006](https://github.com/neovim/neovim/pull/38006) — review the actual landed implementation; specific files depend on what merged.
- [`runtime/lua/vim/ui.lua`](../../raw/neovim/runtime/lua/vim/ui.lua) (`vim.ui.open`)
- [`src/nvim/main.c`](../../raw/neovim/src/nvim/main.c) (CLI argument parsing, `--remote*`)

## File tree

```text
src/nvim/
  main.c
runtime/lua/vim/
  ui.lua
```

## Big-picture references

- [PR #38006](https://github.com/neovim/neovim/pull/38006) and linked discussion.
- Web URI security history: scheme confusion, percent-decoding mismatches, double-slash parsing — the standard rogue's gallery.

## Recent fix / history signal

- [PR #38006](https://github.com/neovim/neovim/pull/38006): the introducing PR. Read maintainer review comments for known limitations.
- Cross-check Vim's `--remote*` family for inherited bugs that became reachable via the URI.

## Audit focus

- Parse the URI grammar from #38006 byte-for-byte. Identify every percent-decoding, normalization, and scheme-stripping step. Each is a candidate for a representation-mismatch bug.
- Find the sink: does `nvim://` reach `:edit`, `:source`, `--remote-send`, or any RPC? `:edit` of attacker file is fine; anything else is a vuln class.
- Re-encoding: if the URI is later re-emitted (logs, status line, window title), check for terminal-escape injection (cross-link to `13-termrequest`).
- Confirm OS-level registration only happens with explicit user consent.

## See Also

- [Terminal TermRequest bridge](13-termrequest.md)
- [Reachability §threat model](../reachability.md#threat-model-justinmk-38985)
