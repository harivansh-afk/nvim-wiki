# `:mkspell` `COMPOUNDFLAG` numeric spellings overflow `AH_KEY_LEN` stack buffers

## Broken invariant

For `FLAG num`, every raw affix-flag spelling that is later copied into
`char key[AH_KEY_LEN]` must either:

- be rejected unless it still fits `AH_KEY_LEN - 1`, or
- be canonicalized to a bounded representation before it is used as a hash key.

That invariant is broken in Neovim's spell affix parser:

- `spell_read_aff()` accepts an arbitrary-length numeric `COMPOUNDFLAG` token and
  stores its raw text in `compflags`
  (`/home/node/workspace/neovim/src/nvim/spellfile.c:2202-2209`).
- End-of-file processing passes that raw text to `process_compflags()`
  (`/home/node/workspace/neovim/src/nvim/spellfile.c:2689-2690`).
- `process_compflags()` parses the numeric value with `get_affitem()`, but then
  copies the original spelling into `char key[AH_KEY_LEN]` with `xmemcpyz()`
  (`/home/node/workspace/neovim/src/nvim/spellfile.c:2865-2871`).
- `xmemcpyz()` writes `len` bytes plus a trailing NUL
  (`/home/node/workspace/neovim/src/nvim/memory.c:240-251`).

So a 17-byte numeric spelling such as `00000000000000001` writes 18 bytes into a
17-byte stack buffer.

The bug is especially clear because the nearby PFX/SFX header path already
expresses the intended policy: new affix headers reject `strlen(items[1]) >=
AH_KEY_LEN` before copying the key
(`/home/node/workspace/neovim/src/nvim/spellfile.c:2329-2348`). That length gate
is missing on the `COMPOUNDFLAG` path.

## Input -> normalization -> policy gate -> sink

1. **Input**: attacker-controlled Myspell source passed to `:mkspell`, for
   example:

   ```aff
   SET UTF-8
   FLAG num
   COMPOUNDFLAG 00000000000000001
   ```

2. **Normalization**:
   - `:mkspell` reaches `mkspell()` and then `spell_read_aff()`
     (`/home/node/workspace/neovim/src/nvim/spellfile.c:4771-4790`,
     `/home/node/workspace/neovim/src/nvim/spellfile.c:5160-5186`).
   - `spell_read_aff()` tokenizes each `.aff` line on whitespace and stores the
     raw item pointer for `items[1]`
     (`/home/node/workspace/neovim/src/nvim/spellfile.c:2049-2102`).
   - `FLAG num` switches the parser into numeric-flag mode
     (`/home/node/workspace/neovim/src/nvim/spellfile.c:2115-2121`).
   - `COMPOUNDFLAG` copies the raw token into `compflags` and appends `+`,
     without any `AH_KEY_LEN` check
     (`/home/node/workspace/neovim/src/nvim/spellfile.c:2202-2209`).
3. **Policy gate**:
   - The codebase already has the right policy for similar consumers: PFX/SFX
     affix headers reject names whose raw spelling is too long for
     `ah_key[AH_KEY_LEN]`
     (`/home/node/workspace/neovim/src/nvim/spellfile.c:2329-2348`).
   - But `COMPOUNDFLAG` skips that raw-length validation entirely and only later
     asks `get_affitem()` whether the text is numerically valid
     (`/home/node/workspace/neovim/src/nvim/spellfile.c:2808-2831`).
   - That numeric parse normalizes the value to integer `1`, but it does not
     bound or canonicalize the original spelling.
4. **Sink**:
   - At end-of-file, `spell_read_aff()` calls `process_compflags()`
     (`/home/node/workspace/neovim/src/nvim/spellfile.c:2689-2690`).
   - `process_compflags()` keeps both representations alive: it parses the flag
     value with `get_affitem()`, but then uses `prevp..p` from the original raw
     string as a hash key and copies it into `char key[AH_KEY_LEN]`
     (`/home/node/workspace/neovim/src/nvim/spellfile.c:2859-2871`).
   - `xmemcpyz(key, prevp, p - prevp)` then writes `17` digits plus a NUL into a
     17-byte stack buffer
     (`/home/node/workspace/neovim/src/nvim/spellfile.c:2842-2843`,
     `/home/node/workspace/neovim/src/nvim/spellfile.c:2870`,
     `/home/node/workspace/neovim/src/nvim/memory.c:246-250`).

This is the broken seam: hostile `.aff` text becomes a numerically-accepted flag,
the raw-spelling length check is missing on the `COMPOUNDFLAG` path, and the raw
token is later copied into a fixed-size stack key buffer.

## Why the invariant should have held

The docs describe `.aff` / `.dic` as external `:mkspell` inputs, not an internal
format:

- `:mkspell` accepts Myspell inputs from `{inname}.aff` + `{inname}.dic`
  (`/home/node/workspace/neovim/runtime/doc/spell.txt:524-527`).
- The `.aff` file documents `FLAG num` as part of the supported external format
  (`/home/node/workspace/neovim/runtime/doc/spell.txt:809-839`).

So Neovim is supposed to safely reject malformed external spell sources, not let
raw flag spellings survive into fixed-size internal buffers.

The local header path already proves the intended invariant: when the same raw
token would be stored in `ah_key`, the parser rejects names that do not fit
`AH_KEY_LEN` (`/home/node/workspace/neovim/src/nvim/spellfile.c:2329-2348`).
`COMPOUNDFLAG` is just the unfixed sibling that bypasses that check.

## Minimal reproducer

### Reproducer files

`/tmp/poc.aff`

```aff
SET UTF-8
FLAG num
COMPOUNDFLAG 00000000000000001
```

`/tmp/poc.dic`

```dic
1
word
```

### Commands

```bash
TMPDIR=$(mktemp -d /tmp/18-spell-affix.XXXXXX)

cat >"$TMPDIR/poc.aff" <<'EOF'
SET UTF-8
FLAG num
COMPOUNDFLAG 00000000000000001
EOF

cat >"$TMPDIR/poc.dic" <<'EOF'
1
word
EOF

nvim --headless -u NONE -i NONE -n \
  "+set nomore" \
  "+mkspell! $TMPDIR/out $TMPDIR/poc" \
  "+qa!"
echo $?
```

### Expected

- `:mkspell` should reject the overlong raw numeric flag spelling with a normal
  parse error, or at minimum fail cleanly without corrupting memory.

### Actual

On this box (`nvim v0.12.1`, release build), the command aborts immediately while
reading the `.aff` file:

```text
Reading affix file /tmp/nvim-spellkeyoverflow.sEX0Jh/poc.aff...*** buffer overflow detected ***: terminated
134
```

I also verified a boundary control:

- `COMPOUNDFLAG 0000000000000001` (16 digits) succeeds and writes a `.spl`.
- `COMPOUNDFLAG 00000000000000001` (17 digits) aborts.

That matches the local source invariant exactly: `AH_KEY_LEN` is 17 bytes total
(`/home/node/workspace/neovim/src/nvim/spellfile.c:386`), and `xmemcpyz()`
always writes the trailing NUL.

## Affected files

- `/home/node/workspace/neovim/src/nvim/spellfile.c:344-386`
  - defines `MAXLINELEN` and `AH_KEY_LEN`.
- `/home/node/workspace/neovim/src/nvim/spellfile.c:2049-2102`
  - tokenizes attacker-controlled `.aff` lines into raw `items[]`.
- `/home/node/workspace/neovim/src/nvim/spellfile.c:2115-2121`
  - enables numeric-flag parsing with `FLAG num`.
- `/home/node/workspace/neovim/src/nvim/spellfile.c:2202-2209`
  - `COMPOUNDFLAG` stores the raw numeric spelling without a length check.
- `/home/node/workspace/neovim/src/nvim/spellfile.c:2329-2348`
  - sibling PFX/SFX header path that already enforces the missing `AH_KEY_LEN`
    bound.
- `/home/node/workspace/neovim/src/nvim/spellfile.c:2689-2690`
  - end-of-file handoff into `process_compflags()`.
- `/home/node/workspace/neovim/src/nvim/spellfile.c:2808-2831`
  - `get_affitem()` numerically accepts the token but does not canonicalize its
    raw spelling.
- `/home/node/workspace/neovim/src/nvim/spellfile.c:2838-2895`
  - `process_compflags()` copies the raw token into `char key[AH_KEY_LEN]`.
- `/home/node/workspace/neovim/src/nvim/memory.c:240-251`
  - `xmemcpyz()` performs the out-of-bounds write by appending a NUL after the
    unbounded copy.
- `/home/node/workspace/neovim/runtime/doc/spell.txt:524-527, 809-839`
  - documents `:mkspell` and `.aff`/`.dic` as external spell-source inputs.

## Impact / severity

This is a real C stack-buffer overflow in a hostile-file parser/generator path.

Reachability is narrower than a plain “open a file and crash” bug because the
victim must run `:mkspell` on attacker-supplied spell sources. But once they do,
the effect is not a clean parse failure:

- attacker-controlled `.aff` text survives tokenization and numeric validation,
- the missing raw-length gate lets the overlong spelling cross into internal
  key-building code,
- and the sink is a fixed-size stack overwrite in C.

I would rate this **medium severity**:

- it is not default-open or modeline-reachable;
- but it is reproducible memory corruption from a single crafted `.aff` line on a
  documented external input boundary;
- hardened builds on this box abort with `*** buffer overflow detected ***`;
- less-hardened builds may turn the same write into a stronger exploitation
  primitive.

The broken invariant is crisp: numeric affix flags are treated as validated once
their integer value parses, but later consumers still key off the attacker-chosen
raw spelling length. That validator/executor drift is what makes the overflow
possible.
