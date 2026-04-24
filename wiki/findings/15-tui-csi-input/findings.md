# CSI key-event subparameters overflow one-element stack buffers

## Broken invariant

CSI key handlers that only expect **one** subparameter must never write past the
caller-provided subparameter capacity.

That invariant is broken in `termkey_interpret_csi_param()`. The handlers for
CSI-SS3 keys, CSI `~` function keys, and CSI-u keys all pass a single `int
subparam` with `nsubparams = 1`, but the helper loops while `length <= capacity`
and then unconditionally stores `subparams[length - 1]` in the epilogue. A
second `:` in attacker-controlled terminal input therefore writes `subparams[1]`
past the one-element stack slot.

## Input -> normalization -> policy gate -> sink

1. **Input**: a hostile host terminal sends a recognized key sequence with an
   extra subparameter, e.g. `ESC [ 65 ; 1:2:3 u` for CSI-u. Equivalent variants
   also exist for arrow/function-key handlers such as `ESC [ 1 ; 1:2:3 A` and
   `ESC [ 15 ; 1:2:3 ~`.
2. **Normalization**:
   - TUI raw input feeds libtermkey from `tk_getkeys()` after
     `handle_raw_buffer()` pushes the bytes into the termkey buffer
     (`src/nvim/tui/input.c:439-466`, `src/nvim/tui/input.c:783-855`).
   - `peekkey_csi_csi()` dispatches recognized CSI commands through the handler
     table (`src/nvim/tui/termkey/driver-csi.c:714-756`).
   - `parse_csi()` deliberately keeps `:` inside a parameter slice because it
     treats every byte `< ';'` as part of the current parameter, so the second
     parameter becomes the slice `1:2:3` instead of being rejected
     (`src/nvim/tui/termkey/driver-csi.c:489-527`).
3. **Policy gate**:
   - The key-event handlers clearly express the intended bound by allocating a
     single `int subparam` and passing `size_t nsubparams = 1`
     (`src/nvim/tui/termkey/driver-csi.c:30-35`,
     `src/nvim/tui/termkey/driver-csi.c:112-116`,
     `src/nvim/tui/termkey/driver-csi.c:188-192`).
   - That should mean “parse at most one subparameter”.
4. **Sink**:
   - `termkey_interpret_csi_param()` violates the bound. After the second `:`,
     `length` becomes `2`, the loop exits, and the epilogue writes
     `subparams[length - 1]`, i.e. `subparams[1]`, past the one-element stack
     buffer (`src/nvim/tui/termkey/driver-csi.c:564-590`).
   - In the real call sites that is an out-of-bounds stack write before key
     dispatch continues.

## Minimal reproducer

### Reproducer file

This harness copies the exact vulnerable helper body from
`src/nvim/tui/termkey/driver-csi.c:545-593` and uses the same caller shape as
`handle_csi_u()` / `handle_csifunc()` / `handle_csi_ss3_full()`:

```c
#include <assert.h>
#include <stddef.h>
#include <stdio.h>

typedef enum {
  TERMKEY_RES_NONE,
  TERMKEY_RES_KEY,
  TERMKEY_RES_EOF,
  TERMKEY_RES_AGAIN,
  TERMKEY_RES_ERROR,
} TermKeyResult;

typedef struct {
  const unsigned char *param;
  size_t length;
} TermKeyCsiParam;

TermKeyResult termkey_interpret_csi_param(TermKeyCsiParam param, int *paramp, int subparams[],
                                          size_t *nsubparams)
{
  if (paramp == NULL) {
    return TERMKEY_RES_ERROR;
  }

  if (param.param == NULL) {
    *paramp = -1;
    if (nsubparams) {
      *nsubparams = 0;
    }
    return TERMKEY_RES_KEY;
  }

  int arg = 0;
  size_t i = 0;
  size_t capacity = nsubparams ? *nsubparams : 0;
  size_t length = 0;
  for (; i < param.length && length <= capacity; i++) {
    unsigned char c = param.param[i];
    if (c == ':') {
      if (length == 0) {
        *paramp = arg;
      } else if (subparams != NULL) {
        subparams[length - 1] = arg;
      }

      arg = 0;
      length++;
      continue;
    }

    assert(c >= '0' && c <= '9');
    arg = (10 * arg) + (c - '0');
  }

  if (length == 0) {
    *paramp = arg;
  } else if (subparams != NULL) {
    subparams[length - 1] = arg;
  }

  if (nsubparams) {
    *nsubparams = length;
  }

  return TERMKEY_RES_KEY;
}

static void caller_shape_like_handle_csi_u(void)
{
  TermKeyCsiParam param = { (const unsigned char *)"1:2:3", 5 };
  int arg = 0;
  int subparam = 0;
  size_t nsubparams = 1;
  (void)termkey_interpret_csi_param(param, &arg, &subparam, &nsubparams);
  printf("arg=%d subparam=%d nsubparams=%zu\n", arg, subparam, nsubparams);
}

int main(void)
{
  caller_shape_like_handle_csi_u();
  return 0;
}
```

### Commands

```bash
cat >/tmp/termkey_csi_subparam_oob.c <<'EOF'
#include <assert.h>
#include <stddef.h>
#include <stdio.h>

typedef enum {
  TERMKEY_RES_NONE,
  TERMKEY_RES_KEY,
  TERMKEY_RES_EOF,
  TERMKEY_RES_AGAIN,
  TERMKEY_RES_ERROR,
} TermKeyResult;

typedef struct {
  const unsigned char *param;
  size_t length;
} TermKeyCsiParam;

TermKeyResult termkey_interpret_csi_param(TermKeyCsiParam param, int *paramp, int subparams[],
                                          size_t *nsubparams)
{
  if (paramp == NULL) {
    return TERMKEY_RES_ERROR;
  }

  if (param.param == NULL) {
    *paramp = -1;
    if (nsubparams) {
      *nsubparams = 0;
    }
    return TERMKEY_RES_KEY;
  }

  int arg = 0;
  size_t i = 0;
  size_t capacity = nsubparams ? *nsubparams : 0;
  size_t length = 0;
  for (; i < param.length && length <= capacity; i++) {
    unsigned char c = param.param[i];
    if (c == ':') {
      if (length == 0) {
        *paramp = arg;
      } else if (subparams != NULL) {
        subparams[length - 1] = arg;
      }

      arg = 0;
      length++;
      continue;
    }

    assert(c >= '0' && c <= '9');
    arg = (10 * arg) + (c - '0');
  }

  if (length == 0) {
    *paramp = arg;
  } else if (subparams != NULL) {
    subparams[length - 1] = arg;
  }

  if (nsubparams) {
    *nsubparams = length;
  }

  return TERMKEY_RES_KEY;
}

static void caller_shape_like_handle_csi_u(void)
{
  TermKeyCsiParam param = { (const unsigned char *)"1:2:3", 5 };
  int arg = 0;
  int subparam = 0;
  size_t nsubparams = 1;
  (void)termkey_interpret_csi_param(param, &arg, &subparam, &nsubparams);
  printf("arg=%d subparam=%d nsubparams=%zu\n", arg, subparam, nsubparams);
}

int main(void)
{
  caller_shape_like_handle_csi_u();
  return 0;
}
EOF

cc -fsanitize=address -fno-omit-frame-pointer -O1 -g \
  /tmp/termkey_csi_subparam_oob.c -o /tmp/termkey_csi_subparam_oob

/tmp/termkey_csi_subparam_oob
```

### Expected

- Extra subparameters beyond the caller-provided capacity should be rejected or
  truncated.
- A caller that passes `nsubparams = 1` should never see a write past its one
  `int subparam`.

### Actual

On this box the command aborts with AddressSanitizer reporting a stack-buffer
overflow on the epilogue store:

```text
==16609==ERROR: AddressSanitizer: stack-buffer-overflow on address ...
WRITE of size 4 at ... thread T0
    #0 ... in termkey_interpret_csi_param /tmp/termkey_csi_subparam_oob.c:58
    #1 ... in caller_shape_like_handle_csi_u /tmp/termkey_csi_subparam_oob.c:74
...
  This frame has 3 object(s):
    [32, 36) 'arg'
    [48, 52) 'subparam' <== Memory access at offset 52 overflows this variable
    [64, 72) 'nsubparams'
```

That is exactly the caller shape used by the real Neovim key-event handlers.

## Affected files

- `src/nvim/tui/input.c:439-466` — libtermkey results are consumed from raw TUI
  input and unknown/recognized CSI handlers are dispatched.
- `src/nvim/tui/input.c:783-855` — raw host-terminal bytes are pushed into the
  libtermkey buffer.
- `src/nvim/tui/termkey/driver-csi.c:25-60` — CSI-SS3 keys parse one event
  subparameter into a single `int subparam`.
- `src/nvim/tui/termkey/driver-csi.c:102-165` — CSI `~` function keys do the
  same.
- `src/nvim/tui/termkey/driver-csi.c:181-220` — CSI-u keys do the same.
- `src/nvim/tui/termkey/driver-csi.c:459-527` — `parse_csi()` preserves `:`
  inside parameter slices.
- `src/nvim/tui/termkey/driver-csi.c:545-593` — `termkey_interpret_csi_param()`
  writes past caller capacity.
- `src/nvim/tui/termkey/driver-csi.c:714-756` — recognized CSI commands are
  dispatched to the vulnerable handlers.

## History / variant signal

The task brief already pointed at `driver-csi.c` as a hot spot with fixed-size
parameter handling history. Upstream fixed the 17th-parameter array write in
`parse_csi()` (`#33868`), but this is the sibling invariant one layer later:
even when the outer 16-parameter cap holds, subparameter parsing still ignores
the caller’s smaller capacity.

The bug is especially easy to miss because it is not in the top-level CSI parser
itself. It appears only after normalization has already turned raw bytes into a
“safe” parameter slice and a downstream handler assumes the slice obeys its own
one-subparam grammar.

## Impact / severity

This is a **default-reachable memory corruption bug** in Neovim’s host-terminal
input path.

An attacker who controls bytes coming from the outer terminal layer can send a
malformed but otherwise recognized CSI-u / CSI-function / CSI-SS3 key sequence
and trigger a stack out-of-bounds write before Neovim finishes decoding the key.
That attacker could be a malicious terminal emulator, a compromised terminal
multiplexer, or another layer that proxies terminal input.

I would rate this **medium severity**:

- it is not file-open-to-RCE;
- the attacker must control host terminal input, not just file contents;
- but it is real C memory corruption on a default-on boundary, not just a logic
  bug or parser oddity.

Without sanitizers the exact consequence depends on compiler and stack layout,
so I am not claiming a stable code-execution exploit here. But the broken
invariant is clear, the write is real, and hostile terminal bytes should never
be able to corrupt adjacent stack state while Neovim decodes a keypress.
