```text
src/
  nvim/
    msgpack_rpc/
      unpacker.c
    grid.c
    ui_client.c
    tui/
      tui.c
    api/
      ui.c
runtime/
  doc/
    remote.txt
```

# Phase 2 - remote UI `grid_line` glyph overflow

## Severity

- Technical severity: high
- Fleet exposure: low
- Why: the sink is real memory corruption from one forged redraw batch, but the victim must be acting as a built-in `--remote-ui` client talking to a malicious server.

## Why this is severe but niche

This is one of the strongest parser-to-memory-corruption bugs in the repo. The message is protocol-shaped, the parse path accepts it, and rendering later crashes on a fixed stack buffer. But the deployment story is much narrower than modeline, ShaDa, or terminal child output bugs because ordinary TUI users do not run as a remote UI client.

So the right mental model is:

- very strong engine bug
- weak default fleet reach

## Technical deep dive

The key mistake is that the redraw fast path treats peer-supplied cell strings as if they were already canonical internal glyphs.

- `unpacker_advance()` upgrades `redraw` notifications into the dedicated redraw-event state machine.
- `unpacker_parse_redraw()` accepts arbitrary `cellsize` and forwards it straight into `schar_from_buf(cellbuf, cellsize)`.
- `schar_from_buf()` explicitly requires `len < MAX_SCHAR_SIZE`.
- later, `tui_raw_line()` stores the resulting `schar_T`, and `print_cell_at_pos()` calls `schar_get()` into `char buf[MAX_SCHAR_SIZE]`.

That means the consumer trusts a producer-side invariant that only honest peers obey. Once an oversized glyph is interned, the later render path copies it back out with no bound check.

## Reproduction on local `nvim v0.12.1`

I reran the real `--remote-ui` proof with a fake upstream server that sends one forged `grid_line` cell containing a 4000-byte string.

Observed result:

```text
script_rc 134
*** buffer overflow detected ***: terminated
```

## Bottom line

This is a real high-severity memory-corruption bug, but it is not one of the broadest user-machine compromise paths because the victim must already be in a niche remote-UI client mode.
