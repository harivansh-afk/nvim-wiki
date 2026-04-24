# Channel transport creation

## Overview

This is a Neovim-specific transport unification layer. [`src/nvim/channel.c`](../../../neovim/src/nvim/channel.c), [`src/nvim/msgpack_rpc/server.c`](../../../neovim/src/nvim/msgpack_rpc/server.c), and [`src/nvim/event/socket.c`](../../../neovim/src/nvim/event/socket.c) bring together jobs, accepted sockets, outbound connections, stdio embed mode, PTYs, and raw streams. It ranks highly because one abstraction error here can affect several externally reachable surfaces at once.

## Relevant options and knobs

- No single editor option governs this layer directly.
- Relevant entrypoints are `jobstart()`, `sockconnect()`, `serverstart()`, `--listen`, and `--embed`.
- The meaningful knobs are transport type (`rpc`, raw, PTY, stdio) and listener address selection, not user-facing option names.

## Relevant files

- [`src/nvim/channel.c`](../../../neovim/src/nvim/channel.c)
- [`src/nvim/msgpack_rpc/server.c`](../../../neovim/src/nvim/msgpack_rpc/server.c)
- [`src/nvim/event/socket.c`](../../../neovim/src/nvim/event/socket.c)

## File tree

```text
src/
  nvim/
    channel.c
    msgpack_rpc/
      server.c
    event/
      socket.c
```

## Big-picture references

- [`runtime/doc/dev_arch.txt`](../../../neovim/runtime/doc/dev_arch.txt): explicitly identifies RPC, jobs, sockets, and PTYs as trust boundaries.
- [`runtime/doc/api.txt`](../../../neovim/runtime/doc/api.txt): notes that localhost TCP sockets are generally less secure than named pipes or Unix sockets.
- [`runtime/doc/vim_diff.txt`](../../../neovim/runtime/doc/vim_diff.txt): helpful because job/channel/terminal behavior diverges sharply from classic Vim.

## Recent fix / history signal

- `bfe9fa0f8e`: stale socket file cleanup fix (#37378).
- `64ce5382bd`: crash on failed `sockconnect()` (#37811).
- `886efcb853`: hang after TCP connect timeout (#37813).

## Audit focus

- Review listener-address parsing and socket lifecycle with the assumption that exposure is unauthenticated by default.
- Check ownership and refcount boundaries when a channel changes state during connect, accept, or teardown.
- Separate RPC channels from raw channels during audit; the abstraction is shared, but the invariants are not identical.
- Pay special attention to stdio/embed and PTY setup, where OS-level handles and editor event queues meet.

## See Also

- [MessagePack-RPC unpacker](10-rpc-unpacker.md)
- [MessagePack-RPC dispatch](11-rpc-dispatch.md)
- [Terminal TermRequest bridge](13-termrequest.md)
