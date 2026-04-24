# RPC and channel surfaces

> Sources: [MessagePack-RPC unpacker](../tasks/10-rpc-unpacker.md); [MessagePack-RPC dispatch](../tasks/11-rpc-dispatch.md); [Channel transport creation](../tasks/12-channel-transports.md)
> Updated: 2026-04-23

## Overview

This area covers the remote control plane from transport creation to byte parsing to request scheduling. Agents should read it as a pipeline: transport setup exposes an endpoint, the unpacker turns bytes into structured messages, and dispatch turns those messages into queued or immediate work inside the editor.

## Covered task briefs

- [MessagePack-RPC unpacker](../tasks/10-rpc-unpacker.md)
- [MessagePack-RPC dispatch](../tasks/11-rpc-dispatch.md)
- [Channel transport creation](../tasks/12-channel-transports.md)

## Key files and entrypoints

- Transport creation and sockets:
  - [src/nvim/channel.c](../../raw/neovim/src/nvim/channel.c)
  - [src/nvim/msgpack_rpc/server.c](../../raw/neovim/src/nvim/msgpack_rpc/server.c)
  - [src/nvim/event/socket.c](../../raw/neovim/src/nvim/event/socket.c)
- Parsing and dispatch:
  - [src/nvim/msgpack_rpc/unpacker.c](../../raw/neovim/src/nvim/msgpack_rpc/unpacker.c)
  - [src/nvim/msgpack_rpc/channel.c](../../raw/neovim/src/nvim/msgpack_rpc/channel.c)
  - [src/nvim/api/private/dispatch.c](../../raw/neovim/src/nvim/api/private/dispatch.c)
  - [src/nvim/api/vim.c](../../raw/neovim/src/nvim/api/vim.c)

## Primary docs and knobs

- [runtime/doc/api.txt](../../raw/neovim/runtime/doc/api.txt)
- [runtime/doc/dev_arch.txt](../../raw/neovim/runtime/doc/dev_arch.txt)
- Exposure knobs: `--listen`, `--embed`, `serverstart()`, `sockconnect()`, `jobstart(..., { rpc: v:true })`

## How agents should navigate this area

1. Start at the entrypoint that creates exposure: socket listener, outbound connection, stdio embed, PTY, or job channel.
2. Follow the flow in order:
   - transport setup
   - msgpack header or redraw parsing
   - request routing and response matching
3. Separate raw channels from RPC channels. They share infrastructure but not identical invariants.
4. Treat close ordering, queue ownership, and fast-event behavior as primary review targets.

## Boundary map

- `channel.c`, `server.c`, and `socket.c` decide how peers connect.
- `unpacker.c` decides what byte streams become valid structured messages.
- `msgpack_rpc/channel.c` decides how those messages are executed or queued.
- Once the issue crosses into terminal jobs or embedded UI flows, continue with [Terminal and TUI surfaces](terminal-and-tui-surfaces.md).

## See Also

- [Terminal and TUI surfaces](terminal-and-tui-surfaces.md)
- [Persistence and parser surfaces](persistence-and-parser-surfaces.md)
- [Task brief navigation](../agents/task-brief-navigation.md)
