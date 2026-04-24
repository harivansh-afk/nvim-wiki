# TUI CSI and raw input parsing

## Overview

This is a Neovim-owned host-terminal input boundary. [`src/nvim/tui/input.c`](../../../neovim/src/nvim/tui/input.c) accepts raw bytes, handles focus and bracketed-paste escapes, grows libtermkey buffers for long sequences, and [`src/nvim/tui/termkey/driver-csi.c`](../../../neovim/src/nvim/tui/termkey/driver-csi.c) parses CSI parameter lists. It ranks highly because arbitrary terminal input arrives here before becoming keys, mode reports, or terminal responses, and this exact area already had out-of-bounds write history.

## Relevant options and knobs

- `'ttimeout'`: controls whether incomplete escape sequences are timed before being interpreted.
- `'ttimeoutlen'`: sets the timeout duration for ambiguous/incomplete terminal sequences.
- `'termpastefilter'`: influences how terminal paste content is filtered before becoming input.

## Relevant files

- [`src/nvim/tui/input.c`](../../../neovim/src/nvim/tui/input.c)
- [`src/nvim/tui/termkey/driver-csi.c`](../../../neovim/src/nvim/tui/termkey/driver-csi.c)

## File tree

```text
src/
  nvim/
    tui/
      input.c
      termkey/
        driver-csi.c
```

## Big-picture references

- [`runtime/doc/vim_diff.txt`](../../../neovim/runtime/doc/vim_diff.txt): documents the simplified `ttimeout` behavior in Nvim’s TUI.
- [`runtime/doc/dev_arch.txt`](../../../neovim/runtime/doc/dev_arch.txt): the architecture-level trust boundary for host terminal bytes becoming editor input.
- [`runtime/doc/api.txt`](../../../neovim/runtime/doc/api.txt): useful for the `termresponse` side of the same overall terminal I/O picture.

## Recent fix / history signal

- `8707ec2644` / `0db89468d7`: termkey out-of-bounds write fix (#33868).
- `884a83049b`: grow termkey internal buffer for large escape sequences (#26309).
- `7a3fef9e34`: out-of-bound access hardening after `snprintf` (#24751).

## Audit focus

- Review the boundary between direct raw handling and libtermkey-mediated parsing; bugs often hide in the handoff, not the parser alone.
- Check fixed-size CSI parameter arrays and any logic that stops parsing after the cap is reached.
- Audit incomplete-sequence handling and timer-driven fallback, because ambiguity itself is part of the attack surface.
- Treat giant escape sequences and paste-like floods as primary test cases, not edge conditions.

## See Also

- [Terminal TermRequest bridge](13-termrequest.md)
- [OSC 52 selection parser](14-osc52-selection.md)
