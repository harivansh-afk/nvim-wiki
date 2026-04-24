# Terminal TermRequest bridge

## Overview

This is a Neovim-specific terminal-control boundary. [`src/nvim/terminal.c`](../../raw/neovim/src/nvim/terminal.c) converts OSC, DCS, and APC sequences from programs running inside `:terminal` into queued `TermRequest` events and related side effects. It ranks highly because untrusted terminal output can cross into autocommands and buffer state, and recent fixes show that lifecycle bugs here are not hypothetical.

## Relevant options and knobs

- `'scrollback'`: influences terminal buffer behavior and therefore the state around pending terminal events.
- `'shell'`: indirectly affects which child process commonly runs inside `:terminal`.
- The real policy knob is user autocommand handling of `TermRequest`, even though that is not represented as an option.

## Relevant files

- [`src/nvim/terminal.c`](../../raw/neovim/src/nvim/terminal.c)
- [`runtime/doc/terminal.txt`](../../raw/neovim/runtime/doc/terminal.txt)

## File tree

```text
src/
  nvim/
    terminal.c
runtime/
  doc/
    terminal.txt
```

## Big-picture references

- [`runtime/doc/terminal.txt`](../../raw/neovim/runtime/doc/terminal.txt): documents `TermRequest` and important OSC use cases such as OSC 7 and OSC 133.
- [`runtime/doc/api.txt`](../../raw/neovim/runtime/doc/api.txt): documents `termresponse` at the UI/API boundary.
- [`runtime/doc/dev_arch.txt`](../../raw/neovim/runtime/doc/dev_arch.txt): places terminal emulation and event-loop interactions in the core architecture map.

## Recent fix / history signal

- `f6ca9262b8`: fix for closing a terminal with pending `TermRequest` (#37227).
- `b40880f88f`: heap UAF if buffer deleted during `TermRequest` (#37612).
- `41a73bce7a` / `2c5fdba076`: pending `TermRequest` `StringBuilder` leak fixes (#39333).

## Audit focus

- Review queueing and replay of pending terminal requests while scrollback is still being applied.
- Check the refcount / destroy sequence when autocommands delete the terminal buffer or close the terminal mid-event.
- Audit the distinction between OSC 8 hyperlink handling and generic `TermRequest` emission.
- Treat user autocommands as adversarial reentrancy sources when reasoning about object lifetime.

## See Also

- [OSC 52 selection parser](14-osc52-selection.md)
- [TUI CSI and raw input parsing](15-tui-csi-input.md)
- [Channel transport creation](12-channel-transports.md)
