# MessagePack-RPC unpacker

## Overview

This is a Neovim-specific remote parser boundary. [`src/nvim/msgpack_rpc/unpacker.c`](../../raw/neovim/src/nvim/msgpack_rpc/unpacker.c) contains `unpacker_parse_header()`, `unpacker_advance()`, and the redraw fast path in `unpacker_parse_redraw()`. It ranks highly because this is the first structured parse of bytes arriving from RPC peers, including special treatment for UI redraw traffic, partial reads, and state-machine recovery.

## Relevant options and knobs

- No ordinary editor option controls this code directly.
- Relevant exposure knobs are `--listen`, `serverstart()`, `sockconnect()`, `jobstart(..., { rpc: v:true })`, and `--embed`.
- Client type and handler lookup also matter operationally even though they are API metadata rather than options.

## Relevant files

- [`src/nvim/msgpack_rpc/unpacker.c`](../../raw/neovim/src/nvim/msgpack_rpc/unpacker.c)
- [`src/nvim/api/private/dispatch.c`](../../raw/neovim/src/nvim/api/private/dispatch.c)

## File tree

```text
src/
  nvim/
    msgpack_rpc/
      unpacker.c
    api/
      private/
        dispatch.c
```

## Big-picture references

- [`runtime/doc/api.txt`](../../raw/neovim/runtime/doc/api.txt): describes msgpack-rpc as the main programmatic control plane and notes automatic API exposure.
- [`runtime/doc/dev_arch.txt`](../../raw/neovim/runtime/doc/dev_arch.txt): frames UI events and RPC transport as core Nvim architecture.
- [`runtime/doc/api.txt`](../../raw/neovim/runtime/doc/api.txt): especially relevant because the build auto-generates dispatch glue for API exposure.

## Recent fix / history signal

- The redraw fast path already had crash fixes around `grid_line` event parsing (#25581).
- RPC/UI lifecycle churn in nearby code keeps showing up in fixes, which is usually a sign that parser state and downstream ownership are tightly coupled.

## Audit focus

- Review header shape validation, method-name length checks, and how EOF / partial-message recovery resets parser state.
- Check the redraw-specific FSM separately from generic msgpack parsing; its assumptions are intentionally different.
- Follow how handler lookup errors are represented and whether malformed input can desynchronize later message processing.
- Treat arena ownership and the `has_grid_line_event` fast path as first-class audit targets, not optimizations beneath notice.

## See Also

- [MessagePack-RPC dispatch](11-rpc-dispatch.md)
- [Channel transport creation](12-channel-transports.md)
