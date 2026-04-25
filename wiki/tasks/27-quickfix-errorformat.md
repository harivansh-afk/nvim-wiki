# Quickfix `'errorformat'` parser

## Overview

`'errorformat'` is a custom mini-language Neovim uses to parse compiler/linter output into the quickfix list. The parser in [`src/nvim/quickfix.c`](../../raw/neovim/src/nvim/quickfix.c) (`qf_parse_line`, `qf_parse_match`) consumes attacker-influenceable bytes from `:cfile`, `:cgetfile`, `:caddfile`, and `:make` output. Listed by `runtime.md` rule 44 as a high-yield missing surface — long parser lifetime, hand-rolled scanf-style format strings, and direct path to filesystem-targeted entries.

## Relevant options and knobs

- `'errorformat'`: the parsing template list. Most ftplugins set it.
- `'makeprg'`, `'grepprg'`: programs whose output feeds the parser.
- `:cfile <name>`, `:cexpr`, `:cbuffer`: command entry points.

## Relevant files

- [`src/nvim/quickfix.c`](../../raw/neovim/src/nvim/quickfix.c) (`qf_parse_line`, `qf_parse_match`, `efm_to_regpat`)
- [`src/nvim/regexp.c`](../../raw/neovim/src/nvim/regexp.c) (efm regex interpretation)

## File tree

```text
src/nvim/
  quickfix.c
  regexp.c
```

## Big-picture references

- [`runtime/doc/quickfix.txt`](../../raw/neovim/runtime/doc/quickfix.txt): `'errorformat'` grammar.
- [`runtime/doc/options.txt`](../../raw/neovim/runtime/doc/options.txt) `*'errorformat'*`.

## Recent fix / history signal

- Vim has a long history of `'errorformat'` parser CVEs and crash bugs (e.g. `efm_to_regpat` overflows). Most pre-2020 Vim CVEs have inherited code in Nvim — verify reachability per [`reachability.md`](../reachability.md#self-check), do not pattern-match.
- [`runtime.md`](runtime.md) rule 44 names this surface explicitly.

## Audit focus

- `efm_to_regpat`: parser converts efm tokens into a regex template. Any unbounded copy or off-by-one is a classic OOB write.
- Path-construction sinks: `qf_parse_line` populates entry filename from regex captures. Does it normalize? Does it allow `..`? Are absolute paths blocked anywhere? (Spoiler: probably not.)
- `:cfile` on attacker-controlled output is the canonical Tier-A trigger — open file → run `:make`/`:cfile` → parser executes.
- Multibyte boundary handling in efm format conversions.

## See Also

- [Statusline mini-language parser](19-statusline-parser.md)
- [Spell affix parser](18-spell-affix-parser.md)
- [Reachability §self-check](../reachability.md#self-check)
