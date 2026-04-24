# Extmark / decoration lifetime

## Overview

Extmarks and decoration providers are Nvim-owned APIs with significant surface and a history of lifetime/ownership bugs. RPC peers, Lua callbacks, and core C all mutate the same kdtree-backed mark store and the decoration provider list. Listed by `runtime.md` rule 44 as missing audit coverage. The class is *attacker controls bytes that become a `Decoration` / extmark, callback fires, original buffer/window torn down or mutated, deref of stale pointer*.

## Relevant options and knobs

- `nvim_buf_set_extmark`, `nvim_buf_del_extmark`: API entry points (RPC- and Lua-reachable).
- `nvim_set_decoration_provider`: registers callbacks that fire during redraw.
- Treesitter highlighter and `vim.lsp.buf.semantic_tokens_full`: heavy real-world users — large bug surface.

## Relevant files

- [`src/nvim/extmark.c`](../../raw/neovim/src/nvim/extmark.c)
- [`src/nvim/decoration.c`](../../raw/neovim/src/nvim/decoration.c) (`decor_check_invalid`, provider invocation)
- [`src/nvim/marktree.c`](../../raw/neovim/src/nvim/marktree.c) (mark tree internals)
- [`src/nvim/api/extmark.c`](../../raw/neovim/src/nvim/api/extmark.c)

## File tree

```text
src/nvim/
  extmark.c
  decoration.c
  marktree.c
  api/extmark.c
```

## Big-picture references

- [`runtime/doc/api.txt`](../../raw/neovim/runtime/doc/api.txt): `nvim_buf_set_extmark`, decoration providers.
- [`runtime.md`](runtime.md) rule 44.

## Recent fix / history signal

- The marktree has had several invalidation/ordering bugs over the years; check the most recent 12 months of commits to `extmark.c`/`decoration.c`/`marktree.c` for "fix:" subjects and audit each adjacent code path for siblings.
- Cross-cutting with [`28-autocmd-deferral`](28-autocmd-deferral.md): decoration providers are conceptually deferred callbacks.

## Audit focus

- A decoration provider callback that grows/shrinks the buffer mid-redraw — does the redraw loop hold pointers/iterators across the call?
- Extmark deletion concurrent with redraw: pointer to a freed mark walked by the iterator?
- API surface: can an RPC peer create extmarks faster than the marktree can be torn down on `:bwipeout`?
- Treesitter highlighter is the highest-volume caller — bugs there compound across every buffer with active highlighting.

## See Also

- [Autocmd deferred execution and reentrancy](28-autocmd-deferral.md)
- [MessagePack-RPC dispatch](11-rpc-dispatch.md)
- [Treesitter language loader](23-treesitter-language-loader.md)
