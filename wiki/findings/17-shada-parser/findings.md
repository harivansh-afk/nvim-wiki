# ShaDa variable names reach a write-side stack buffer overflow

## Broken invariant

A variable name accepted from a ShaDa file must still satisfy the write-side
serializer's fixed-buffer precondition before Neovim turns it back into a ShaDa
item.

That invariant is broken across the read/apply/write chain:

- `shada_read_next_item()` accepts an arbitrary MessagePack string length for
  `kSDItemVariable` names as long as the overall item is below the generic item
  size limit (`/home/node/workspace/neovim/src/nvim/shada.c:3118-3138`,
  `/home/node/workspace/neovim/src/nvim/shada.c:3367-3408`).
- startup restore applies that name to `g:` with `var_set_global()`, and the
  underlying `set_var()` path validates only the variable-name character class,
  not its length (`/home/node/workspace/neovim/src/nvim/shada.c:1072-1076`,
  `/home/node/workspace/neovim/src/nvim/eval.c:6332-6338`,
  `/home/node/workspace/neovim/src/nvim/eval/vars.c:2877-2886`,
  `/home/node/workspace/neovim/src/nvim/eval/vars.c:3050-3061`).
- on exit, `'shada'` includes `!` by default, so all-uppercase globals are
  serialized back out (`/home/node/workspace/neovim/runtime/doc/options.txt:5426-5441`,
  `/home/node/workspace/neovim/src/nvim/shada.c:2239-2273`,
  `/home/node/workspace/neovim/src/nvim/shada.c:2326-2389`,
  `/home/node/workspace/neovim/src/nvim/eval.c:6316-6329`).
- `shada_pack_entry()` then copies `"variable g:" + name + "\0"` into
  `char vardesc[256]` with `memcpy(..., varname.size + 1)`, so any accepted name
  longer than 244 bytes overflows the stack buffer
  (`/home/node/workspace/neovim/src/nvim/shada.c:1387-1396`).

`sizeof("variable g:") - 1` is 11, so the copy fits only when
`11 + varname.size + 1 <= 256`, i.e. `varname.size <= 244`.

## Input -> normalization -> policy gate -> sink

1. **Input**: a crafted ShaDa file contains a `kSDItemVariable` entry whose
   payload is `[ <245-byte all-uppercase name>, 0 ]`.
2. **Normalization**:
   - `shada_read_next_item()` reads the outer `(type, timestamp, length)` tuple
     and only rejects absurdly large items or entries above the configured `s`
     limit (`/home/node/workspace/neovim/src/nvim/shada.c:3118-3173`).
   - for `kSDItemVariable`, it reads the name with `unpack_string()` and copies
     it verbatim into `entry->data.global_var.name` via `xmemdupz(name.data,
     name.size)` (`/home/node/workspace/neovim/src/nvim/shada.c:3367-3383`).
3. **Policy gate**:
   - the restore path calls `var_set_global(name, value)` for variable entries
     (`/home/node/workspace/neovim/src/nvim/shada.c:1072-1076`).
   - `set_var()` / `valid_varname()` enforce only identifier syntax; they do not
     cap length (`/home/node/workspace/neovim/src/nvim/eval.c:6332-6338`,
     `/home/node/workspace/neovim/src/nvim/eval/vars.c:2811-2886`,
     `/home/node/workspace/neovim/src/nvim/eval/vars.c:3050-3061`).
   - `var_flavour()` classifies all-uppercase names as `VAR_FLAVOUR_SHADA`, and
     the default `'shada'` value includes `!`, so `shada_write()` serializes the
     imported variable again on exit (`/home/node/workspace/neovim/src/nvim/eval.c:6316-6329`,
     `/home/node/workspace/neovim/runtime/doc/options.txt:5426-5441`,
     `/home/node/workspace/neovim/src/nvim/shada.c:2239-2273`,
     `/home/node/workspace/neovim/src/nvim/shada.c:2326-2389`).
4. **Sink**:
   - `shada_pack_entry()` builds a diagnostic description buffer for
     `encode_vim_to_msgpack()` using:
     - `char vardesc[256] = "variable g:";`
     - `memcpy(&vardesc[sizeof("variable g:") - 1], varname.data,
       varname.size + 1);`
   - a 245-byte name therefore writes one byte past `vardesc`, and longer names
     write further past it (`/home/node/workspace/neovim/src/nvim/shada.c:1391-1396`).

## Minimal reproducer

### Reproducer file

```python
from pathlib import Path

name = b'A' * 245
payload = bytes([0x92, 0xD9, len(name)]) + name + bytes([0x00])
blob = bytes([
    0x01, 0x00, 0x00,        # header item: type=1, timestamp=0, length=0
    0x06, 0x00, 0xCC, len(payload),
]) + payload                 # variable item: type=6, timestamp=0, length=249

Path('/tmp/shada-overflow.shada').write_bytes(blob)
```

This is a 256-byte ShaDa file with two items:

- a minimal zero-length header item; and
- a variable item whose name is 245 uppercase `A` bytes and whose value is the
  integer `0`.

### Commands

```bash
cat >/tmp/17-shada-parser-repro.py <<'EOF'
from pathlib import Path

name = b'A' * 245
payload = bytes([0x92, 0xD9, len(name)]) + name + bytes([0x00])
blob = bytes([
    0x01, 0x00, 0x00,
    0x06, 0x00, 0xCC, len(payload),
]) + payload

Path('/tmp/shada-overflow.shada').write_bytes(blob)
EOF

python3 /tmp/17-shada-parser-repro.py

# Control run: prove the variable is accepted on read, then disable shada before exit.
XDG_STATE_HOME=/tmp/nvim-state-read   nvim --headless -u NONE -i /tmp/shada-overflow.shada   "+set shada="   "+lua io.stdout:write(tostring(vim.g[string.rep('A', 245)]))"   +qall

# Trigger run: keep default shada writeback enabled.
XDG_STATE_HOME=/tmp/nvim-state-crash   nvim --headless -u NONE -i /tmp/shada-overflow.shada +qall
```

### Expected

- Reading a ShaDa variable name from disk should not make later `:wshada` /
  exit-time writeback unsafe.
- If Neovim accepts the name at all, writeback should either serialize it
  safely or reject it cleanly.
- A one-byte increase from 244 to 245 should not turn a persisted variable name
  into a stack overwrite.

### Actual

On this box (`nvim v0.12.1`, release build):

- the control run prints `0`, proving the 245-byte uppercase variable name is
  accepted from the malicious ShaDa file and restored into `g:`;
- the trigger run aborts during exit-time ShaDa writeback with glibc's stack
  protection:

```text
*** buffer overflow detected ***: terminated
```

- the crashing command exits with status `134`;
- the 244-byte boundary case exits cleanly with status `0`, matching the
  `vardesc[256]` threshold calculation.

## Affected files

- `/home/node/workspace/neovim/src/nvim/shada.c:1072-1076`
  - startup restore applies ShaDa variable items with `var_set_global()`.
- `/home/node/workspace/neovim/src/nvim/shada.c:1387-1396`
  - `shada_pack_entry()` overflows `char vardesc[256]` while preparing the
    variable description string.
- `/home/node/workspace/neovim/src/nvim/shada.c:2239-2273`
  - default writeback policy enables variable serialization whenever `'shada'`
    contains `!`.
- `/home/node/workspace/neovim/src/nvim/shada.c:2326-2389`
  - `shada_write()` iterates current `VAR_FLAVOUR_SHADA` globals and forwards
    them to the vulnerable serializer.
- `/home/node/workspace/neovim/src/nvim/shada.c:3118-3173`
  - the generic item reader only enforces coarse item-length policy before
    variable-specific parsing.
- `/home/node/workspace/neovim/src/nvim/shada.c:3367-3408`
  - `kSDItemVariable` parsing copies the attacker-controlled name with no
    per-name length bound.
- `/home/node/workspace/neovim/src/nvim/eval.c:6316-6338`
  - `var_flavour()` classifies all-uppercase names as ShaDa variables and
    `var_set_global()` forwards accepted names into normal variable creation.
- `/home/node/workspace/neovim/src/nvim/eval/vars.c:2811-2886`
  - `set_var_const()` allocates the variable name after syntax checks only.
- `/home/node/workspace/neovim/src/nvim/eval/vars.c:3050-3061`
  - `valid_varname()` enforces character validity, not length.
- `/home/node/workspace/neovim/runtime/doc/options.txt:5426-5441`
  - `'shada'` is enabled by default, auto-read on startup, auto-written on exit,
    and `!` persists uppercase globals.

## Impact / severity

This is a **default-on local file parsing memory corruption bug** on Neovim's
persisted-state boundary.

The attacker primitive is narrow but real: get a crafted `.shada` file read by
Neovim, or convince the victim to point `-i` / `'shadafile'` at one. Once the
malicious variable is imported, ordinary exit-time ShaDa writeback is enough to
hit a fixed-size stack overflow in C. No plugin, modeline, or niche feature is
required; this is core session-state handling.

I would rate it **moderate severity**:

- it is a real stack overflow in release builds, not just a debug assertion;
- it is on an automatically processed file format that Neovim reads on startup
  and writes on exit by default;
- exploitability is weaker than a pure open-file bug because the attacker needs
  control of the ShaDa file path or contents rather than arbitrary buffer text;
- but once that boundary is crossed, the sink is memory corruption, and this box
  aborts reliably with `*** buffer overflow detected ***`.

The broken invariant is representation drift across phases: a name that is
considered acceptable parser output and acceptable editor state is later reused
as if it had already been proven to fit a 256-byte serializer scratch buffer.
