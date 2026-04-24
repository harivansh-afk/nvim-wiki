# MessagePack-RPC dispatch

## Overview

This is the Neovim-specific step where parsed RPC messages become work scheduled inside the editor. [`src/nvim/msgpack_rpc/channel.c`](../../raw/neovim/src/nvim/msgpack_rpc/channel.c) contains `parse_msgpack()`, `handle_request()`, and `request_event()`. It ranks highly because request/response association, fast-event behavior, queue routing, and object lifetime all meet here. If this layer is wrong, malformed or malicious peers do not just crash parsing—they can distort control flow.

## Relevant options and knobs

- No single user option controls this path directly.
- `nvim_set_client_info()` metadata changes protocol assumptions for response matching, which makes client type a security-relevant “knob”.
- Fast vs deferred handler classification is an internal knob with user-visible reentrancy impact.

## Relevant files

- [`src/nvim/msgpack_rpc/channel.c`](../../raw/neovim/src/nvim/msgpack_rpc/channel.c)
- [`src/nvim/api/vim.c`](../../raw/neovim/src/nvim/api/vim.c)

## File tree

```text
src/
  nvim/
    msgpack_rpc/
      channel.c
    api/
      vim.c
```

## Big-picture references

- [`runtime/doc/api.txt`](../../raw/neovim/runtime/doc/api.txt): documents request ordering, `RPC only` APIs, and the control-plane role of msgpack-rpc.
- [`runtime/doc/dev_arch.txt`](../../raw/neovim/runtime/doc/dev_arch.txt): describes the event loop and fast-event constraints that shape dispatch behavior.
- [`runtime/doc/api.txt`](../../raw/neovim/runtime/doc/api.txt): also matters because API methods like `nvim_exec_lua()` become high-value post-connect primitives once dispatch works.

## Recent fix / history signal

- `5d66ef188f` / `eee2d10fd2`: UILeave ordering fix on channel close (#38846).
- Recent RPC fixes repeatedly cluster around close ordering, write failures, and client metadata assumptions rather than pure parse correctness.

## Audit focus

- Review response matching for `msgpack-rpc` versus fallback client types; protocol ambiguity here is security-relevant, not cosmetic.
- Trace arena ownership transfer from unpacker to queued request objects and then into handler return paths.
- Check how fast handlers, resize events, and immediate execution differ from normal queued requests.
- Treat any path that mixes close/error handling with queued events as a likely regression hotspot.

## See Also

- [MessagePack-RPC unpacker](10-rpc-unpacker.md)
- [Channel transport creation](12-channel-transports.md)
- [Terminal TermRequest bridge](13-termrequest.md)
