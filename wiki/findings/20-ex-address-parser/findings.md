# Ex absolute address truncation reaches live command sinks

## Broken invariant

Absolute decimal Ex addresses must be rejected if they cannot be represented as `linenr_T` before any later range validation or command-specific normalization runs.

Today that invariant is broken in `get_address()`: oversized decimal text is parsed with `getdigits()` and then narrowed straight into `linenr_T` (`int32_t`) without an `INT32_MAX` check. Later policy code validates and normalizes the truncated value, not the original attacker-controlled number.

## Input -> normalization -> policy gate -> sink

1. **Input**: an oversized absolute address such as `99999999999999999999999999999999delete` or `4294967298delete`.
2. **Normalization**: `get_address()` takes the absolute-number path and does `lnum = (linenr_T)getdigits(&cmd, false, 0);` (`src/nvim/ex_docmd.c:3706-3708`).
   - `getdigits()` returns the default value on parse overflow (`src/nvim/charset.c:1116-1123`).
   - `linenr_T` is only `int32_t` (`src/nvim/pos_defs.h:5-15`), so values that survive `intmax_t` parsing are still narrowed without bounds checks.
3. **Policy gate**: later validation only sees the narrowed `linenr_T`.
   - `invalid_range()` rejects negative or too-large lines, but accepts `0` (`src/nvim/ex_docmd.c:3840-3927`).
   - `correct_range()` then coerces `0` to `1` for ordinary commands (`src/nvim/ex_docmd.c:3931-3940`).
4. **Sink**: the command executor uses the coerced/wrapped line for the real command (`src/nvim/ex_docmd.c:2267-2303`), so destructive commands such as `:delete`, `:move`, `:copy`, `:write`, and any other range-taking command run on the wrong line instead of failing.

## Why this violates the documented contract

Neovim's Ex range docs say: `The {number} must be between 0 and the number of lines in the file.` (`runtime/doc/cmdline.txt:856-859`).

The current parser does not enforce that on absolute decimal addresses. It silently maps huge values into different in-range numbers.

## Minimal reproducer

### Reproducer file

```text
alpha
beta
gamma
```

### Commands

```bash
TMP=$(mktemp)
printf 'alpha\nbeta\ngamma\n' > "$TMP"

# Parser artifact: huge absolute address is accepted as range 0 instead of rejected.
cat > /tmp/exaddr_parse_only.lua <<'EOF2'
print(vim.json.encode(vim.api.nvim_parse_cmd('99999999999999999999999999999999delete', {})))
vim.cmd('qa!')
EOF2
/home/node/.nix-profile/bin/nvim --clean -u NONE -i NONE -n --headless -S /tmp/exaddr_parse_only.lua

# Execution artifact: same address deletes line 1 instead of failing.
/home/node/.nix-profile/bin/nvim --clean -u NONE -i NONE -n --headless "$TMP" \
  +"99999999999999999999999999999999delete | %print | qall!"
```

### Expected

- The executed Ex command should fail with an invalid/out-of-range error and leave the buffer unchanged.

### Actual

- `nvim_parse_cmd()` returns a parsed command with `"range":[0]`, which shows the overflow already collapsed to zero inside the parser.
- The Ex command still succeeds and prints:

```text
beta
gamma
```

That output means the huge address was normalized to `0`, then coerced to line `1`, and `:delete` removed `alpha`.

### This box: line-targeting wrap variant

On this Linux/x86_64 box, values that fit in `intmax_t` but not `int32_t` also wrap through the same seam. For example:

```bash
TMP=$(mktemp)
printf 'alpha\nbeta\ngamma\n' > "$TMP"
/home/node/.nix-profile/bin/nvim --clean -u NONE -i NONE -n --headless "$TMP" \
  +"4294967298delete | %print | qall!"
```

Actual output:

```text
alpha
gamma
```

So `4294967298` is treated as line `2` instead of being rejected.

## Affected files

- `src/nvim/ex_docmd.c:3706-3708` — absolute decimal addresses are parsed with unbounded `getdigits()` and narrowed to `linenr_T`.
- `src/nvim/ex_docmd.c:2267-2303` — command execution validates and then uses the narrowed range.
- `src/nvim/ex_docmd.c:3840-3927` — `invalid_range()` validates the truncated representation.
- `src/nvim/ex_docmd.c:3931-3940` — `correct_range()` turns `0` into `1` for most commands.
- `src/nvim/charset.c:1116-1123` — `getdigits()` returns the default value on overflow.
- `src/nvim/pos_defs.h:5-15` — `linenr_T` is `int32_t` and `MAXLNUM` is `INT32_MAX`.
- `runtime/doc/cmdline.txt:856-859` — documented numeric-range contract.

## History / variant signal

Recent fixes already restored the same invariant for neighboring paths:

- `3ed800e998` — bounded relative address/count parsing and overflow checks.
- `809b05bf27` — fixed the relative `lnum + n` overflow guard in `get_address()`.
- `3700d94c6f` — ensured E1247 parse failures poison parser state instead of continuing.

The remaining absolute-decimal path is the obvious unfixed sibling: the fixed offset/count code uses bounded `int32` parsing, but the absolute-number branch still does not.

## Impact / severity

This is an integrity bug in a front-door parser that sits before a large set of Ex sinks.

If a plugin, RPC bridge, or helper script forwards attacker-controlled numeric addresses into Ex and relies on Neovim to reject out-of-range values, that assumption is wrong. An attacker can supply oversized decimals that are normalized to `0` or to a small wrapped positive line, redirecting any range-taking command onto unintended content.

That includes destructive or externally visible sinks such as `:delete`, `:move`, `:copy`, `:write`, and commands that execute over a selected range. I would rate this as **medium severity**: it is not direct code execution by itself, but it is a reproducible parser-to-sink integrity violation in a core command surface, and it defeats a natural editor-side bounds-checking assumption.
