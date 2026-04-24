# Terminal and TUI surfaces

> Sources: [Terminal TermRequest bridge](../tasks/13-termrequest.md); [OSC 52 selection parser](../tasks/14-osc52-selection.md); [TUI CSI and raw input parsing](../tasks/15-tui-csi-input.md)
> Updated: 2026-04-23

## Overview

This area covers both directions of terminal I/O. On one side, programs running inside `:terminal` emit OSC, DCS, and APC sequences that can become `TermRequest` events or clipboard actions. On the other side, the host terminal sends raw bytes and CSI sequences that the TUI stack turns into keys, mode reports, or paste data.

## Covered task briefs

- [Terminal TermRequest bridge](../tasks/13-termrequest.md)
- [OSC 52 selection parser](../tasks/14-osc52-selection.md)
- [TUI CSI and raw input parsing](../tasks/15-tui-csi-input.md)

## Key files and entrypoints

- Child-output side:
  - [src/nvim/terminal.c](../../raw/neovim/src/nvim/terminal.c)
  - [src/nvim/vterm/state.c](../../raw/neovim/src/nvim/vterm/state.c)
- Host-input side:
  - [src/nvim/tui/input.c](../../raw/neovim/src/nvim/tui/input.c)
  - [src/nvim/tui/termkey/driver-csi.c](../../raw/neovim/src/nvim/tui/termkey/driver-csi.c)
- Supporting docs:
  - [runtime/doc/terminal.txt](../../raw/neovim/runtime/doc/terminal.txt)
  - [runtime/doc/api.txt](../../raw/neovim/runtime/doc/api.txt)

## Primary docs and knobs

- [runtime/doc/dev_arch.txt](../../raw/neovim/runtime/doc/dev_arch.txt)
- [runtime/doc/vim_diff.txt](../../raw/neovim/runtime/doc/vim_diff.txt)
- High-value knobs: `'scrollback'`, `'shell'`, `'ttimeout'`, `'ttimeoutlen'`, `'termpastefilter'`

## How agents should navigate this area

1. Decide whether the input comes from the terminal child or the host terminal.
2. For child-output issues, start in `terminal.c` and then narrow into `vterm/state.c` when the bug is sequence-specific.
3. For host-input issues, start in `tui/input.c` and then inspect `driver-csi.c` for CSI parameter parsing.
4. Treat callbacks and autocommands as reentrancy points on both sides of the boundary.

## Boundary map

- `terminal.c` turns terminal control output into editor-visible state and events.
- `vterm/state.c` holds smaller parsers such as OSC 52 that still carry high risk.
- `tui/input.c` and termkey turn host terminal bytes into editor input.
- If the issue originates at job or socket setup rather than terminal parsing itself, continue with [RPC and channel surfaces](rpc-and-channel-surfaces.md).

## See Also

- [RPC and channel surfaces](rpc-and-channel-surfaces.md)
- [Task brief navigation](../agents/task-brief-navigation.md)
- [Neovim project goals and layout](../neovim/project-goals-and-layout.md)
