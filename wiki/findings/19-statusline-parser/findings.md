# Right-aligned statusline groups reuse byte counts as fill iterations

## Broken invariant

Right-aligned statusline groups (`%(...%)`) must keep two different quantities
separate:

1. the number of **display cells** still missing from the group, and
2. the number of **bytes** needed to shift the already-rendered group text when
   the fill character is multibyte.

`build_stl_str_hl()` breaks that invariant in the right-aligned group-padding
branch:

- it first converts missing cells into a byte count with
  `group_len = (min_group_width - group_len) * schar_len(fillchar)`, then
- reuses that byte count as the loop bound for `schar_get_adv(&t, fillchar)`.

That means each missing display cell causes `schar_len(fillchar)` *extra* fill
copies, and each copy writes `schar_len(fillchar)` bytes. The result is
renderer-owned memory corruption, not just a cosmetic width bug.

Affected code:

- `/home/node/workspace/neovim/src/nvim/statusline.c:1157-1185`
- `/home/node/workspace/neovim/src/nvim/grid.c:158-181`

This is the same byte-vs-cell invariant family as the recent `#38102`
statusline fix, but in the grouped right-alignment branch instead of the
separator-padding branch.

## Input -> normalization -> policy gate -> sink

1. **Input**
   - Attacker-controlled statusline text uses a right-aligned group such as
     `%50(x%)`.
   - The fill character is a supported multibyte single-width glyph via
     `'fillchars'`, e.g. `stl:𐍈`.
   - Neovim documents both surfaces as supported:
     - `'fillchars'` supports single-byte and multibyte characters as long as
       they are not double-width
       (`/home/node/workspace/neovim/runtime/doc/options.txt:2848-2898`).
     - `%(...)` is the documented grouped-width syntax for statusline formats
       (`/home/node/workspace/neovim/runtime/doc/options.txt:6346-6420`).

2. **Normalization**
   - Setting `'statusline'` reaches
     `did_set_statustabline_rulerformat()` and then `check_stl_option()`
     (`/home/node/workspace/neovim/src/nvim/optionstr.c:1846-1897`,
     `/home/node/workspace/neovim/src/nvim/optionstr.c:290-355`).
   - During redraw, `win_redr_custom()` resolves the current statusline
     fillchar with `fillchar_status()`, copies the format string, and calls
     `build_stl_str_hl()`
     (`/home/node/workspace/neovim/src/nvim/statusline.c:216-223`,
     `/home/node/workspace/neovim/src/nvim/statusline.c:271-338`).
   - Inside `build_stl_str_hl()`, the parser reads the group min width, clamps
     it to 50, and records the group metadata
     (`/home/node/workspace/neovim/src/nvim/statusline.c:1196-1303`).

3. **Policy gate**
   - `check_stl_option()` only validates `%`-grammar structure. `%50(x%)`
     passes because it is a balanced, documented group expression
     (`/home/node/workspace/neovim/src/nvim/optionstr.c:290-355`).
   - Nothing in the setter or validator re-checks whether the chosen fill
     character changes the byte arithmetic of grouped right-alignment.

4. **Sink**
   - When `%)` closes a right-aligned group, the post-processing path computes a
     byte shift for `memmove()` and then reuses that byte count as the number of
     fill iterations
     (`/home/node/workspace/neovim/src/nvim/statusline.c:1157-1185`).
   - `schar_get_adv()` copies the full UTF-8 encoding every iteration
     (`/home/node/workspace/neovim/src/nvim/grid.c:158-171`), so the loop
     overwrites the shifted text and, after enough repeats, overruns the output
     buffer itself.
   - In the normal redraw path that output buffer is a stack
     `char buf[MAXPATHL]` in `win_redr_custom()`
     (`/home/node/workspace/neovim/src/nvim/statusline.c:216-223`).
   - The same underlying bug is also reachable via the public
     `nvim_eval_statusline()` API, which validates the format string with
     `check_stl_option()` and then passes an arena buffer into
     `build_stl_str_hl()`
     (`/home/node/workspace/neovim/src/nvim/api/vim.c:2207-2225`,
     `/home/node/workspace/neovim/src/nvim/api/vim.c:2298-2309`).

## Minimal reproducer

The current host reports `/usr/bin/nvim` as `NVIM v0.12.0-dev`.

### Quick proof of corruption

This one-item probe shows the bug before the full crash: the grouped right
alignment should yield width 5, but the renderer returns width 8 and corrupts
the trailing UTF-8 sequence.

```bash
/usr/bin/nvim --clean --headless \
  "+lua print(vim.fn.strdisplaywidth('𐍈')); local r=vim.api.nvim_eval_statusline('%5(x%)', {maxwidth=80, fillchar='𐍈'}); print(vim.json.encode(r))" \
  +qa
```

Expected:

- the first line is `1`, proving `𐍈` is a single-width glyph on this host;
- the second line should describe a width-5 string equivalent to `𐍈𐍈𐍈𐍈x`.

Actual:

```text
1
{"str":"𐍈𐍈𐍈𐍈<f0>","width":8}
```

The final `x` is overwritten by the lead byte of another fill glyph.

### Crash reproducer on the real option surface

#### Reproducer file

```vim
set laststatus=2
set fillchars=stl:𐍈
let &statusline = repeat('%50(x%)', 18)
redraw!
qa!
```

#### Commands

```bash
cat >/tmp/19-statusline-parser-repro.vim <<'EOF'
set laststatus=2
set fillchars=stl:𐍈
let &statusline = repeat('%50(x%)', 18)
redraw!
qa!
EOF

/usr/bin/nvim --clean --headless -u NONE -S /tmp/19-statusline-parser-repro.vim
```

Expected:

- Neovim should render or at worst truncate the statusline and exit cleanly.

Actual:

```text
*** stack smashing detected ***: terminated
```

On this host the process aborts with a non-zero exit code during `redraw!`.

### Additional verifier: same bug via the public API

```bash
/usr/bin/nvim --clean --headless \
  "+lua local fmt=string.rep('%50(x%)', 18); local ok,res=pcall(vim.api.nvim_eval_statusline, fmt, {maxwidth=4000, fillchar='𐍈'}); print('ok='..tostring(ok))" \
  +qa
```

Expected:

- `ok=true` and a clean exit.

Actual:

```text
double free or corruption (!prev)
```

That shows the same primitive also corrupts heap-backed output buffers, not
just the normal redraw stack buffer.

## Affected files with line references

- `/home/node/workspace/neovim/runtime/doc/options.txt:2848-2898`
  (`'fillchars'` supports multibyte single-width glyphs)
- `/home/node/workspace/neovim/runtime/doc/options.txt:6346-6420`
  (statusline width/group syntax, including `%(...)`)
- `/home/node/workspace/neovim/src/nvim/optionstr.c:290-355`
  (`check_stl_option()` accepts the grouped format)
- `/home/node/workspace/neovim/src/nvim/optionstr.c:1846-1897`
  (`'statusline'` setter uses `check_stl_option()`)
- `/home/node/workspace/neovim/src/nvim/statusline.c:216-223`
  (`win_redr_custom()` stack `buf[MAXPATHL]`)
- `/home/node/workspace/neovim/src/nvim/statusline.c:271-338`
  (`fillchar_status()` + `build_stl_str_hl()` call)
- `/home/node/workspace/neovim/src/nvim/statusline.c:1157-1185`
  (vulnerable right-aligned group padding branch)
- `/home/node/workspace/neovim/src/nvim/statusline.c:1196-1303`
  (group width parsing and clamp)
- `/home/node/workspace/neovim/src/nvim/api/vim.c:2207-2225`
  (`nvim_eval_statusline()` validation path)
- `/home/node/workspace/neovim/src/nvim/api/vim.c:2298-2309`
  (`nvim_eval_statusline()` passes the output buffer into `build_stl_str_hl()`)
- `/home/node/workspace/neovim/src/nvim/grid.c:158-181`
  (`schar_get_adv()` / `schar_len()` copy full multibyte glyphs)

## Impact / severity

This is a real memory-corruption bug in a core C rendering path:

- the normal statusline redraw surface can smash a stack buffer;
- the public `nvim_eval_statusline()` API can corrupt heap-backed buffers; and
- both paths are reachable with documented statusline syntax plus documented
  multibyte fill characters.

The exploit primitive is not a mere display glitch. The renderer loses track of
where valid output ends, overwrites nearby memory with attacker-controlled UTF-8
lead/trailing bytes, and eventually aborts under modern hardening. In less
hardened builds this is classic memory-unsafety territory.

I would rate this as **high-severity memory corruption** in the statusline
subsystem, with at minimum a reliable crash/DoS against any context that
evaluates attacker-controlled statusline formats or exposes the
`nvim_eval_statusline()` API to untrusted peers.
