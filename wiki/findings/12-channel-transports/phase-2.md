```text
src/
  nvim/
    eval/
      funcs.c
    channel.c
    event/
      socket.c
    msgpack_rpc/
      channel.c
    api/
      private/
        dispatch.c
```

# Phase 2 - sandboxed `sockconnect()` RPC escape

## Severity

- Technical severity: high
- Fleet exposure: medium
- Why: the missing gate is in a common API entrypoint, and once it is reached from any sandboxed or secure evaluation path the attacker gets the full trusted RPC surface.

## Why this is more important than it first looks

This bug is not just another transport bug. It is a capability escalator.

If an attacker can reach **any** sandboxed evaluation context that still permits `sockconnect(..., {'rpc': v:true})`, the rest of Neovim's RPC surface becomes reachable from the outside. The remote peer does not need `rpcrequest()` or `rpcnotify()`. It can initiate the first request itself.

That means this finding amplifies other boundary mistakes. A weak "only sandboxed code ran" bug stops being contained once `sockconnect()` can mint a trusted RPC peer.

## Technical deep dive

The critical chain is short and direct:

- `f_sockconnect()` parses `opts.rpc` but never calls `check_secure()`.
- `channel_connect(..., rpc=true, ...)` opens the socket and hands the channel to `rpc_start()`.
- `rpc_start()` marks the channel RPC-capable and starts msgpack reception.
- `parse_msgpack()` and the normal dispatch table then accept attacker-initiated API calls such as `nvim_command()` and `nvim_exec_lua()`.

The real boundary failure is that Neovim protects the *RPC send helpers* but forgets to protect the *transport creation helper* that is enough to let the peer drive the session from the other side.

## Reproduction on local `nvim v0.12.1`

I reproduced the exact escape shape with a sandboxed call that opens an RPC connection back to a local test server.

Observed result:

```text
{'rc': 0, 'accepted': True, 'marker_exists': True, 'response': '940101c0c0'}
```

`marker_exists: True` means the remote peer successfully invoked `nvim_command()` and caused a side effect from a sandboxed starting point.

## Bottom line

This is one of the strongest findings in the repo because it turns a restricted script context into a full editor-control channel. The precondition is not "rare transport mode", it is "any sandboxed path that can still call `sockconnect`".
