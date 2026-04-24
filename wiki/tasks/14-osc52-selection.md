# OSC 52 selection parser

## Overview

This is a narrow Neovim-owned parser inside the terminal stack. [`src/nvim/vterm/state.c`](../../../neovim/src/nvim/vterm/state.c) implements `osc_selection()`, which parses OSC 52 clipboard / selection requests, decodes base64 fragments, tracks partial state across fragments, and forwards decoded data into callbacks. It ranks highly because it is externally driven, stateful, and adjacent to clipboard behavior—exactly the kind of small parser that can hide high-impact mistakes.

## Relevant options and knobs

- No direct editor option controls `osc_selection()` itself.
- The relevant “knob” is whether the terminal stack has selection callbacks configured, especially clipboard-oriented ones.
- This path is distinct from general `clipboard` settings; OSC 52 behavior is wired through the embedded terminal path.

## Relevant files

- [`src/nvim/vterm/state.c`](../../../neovim/src/nvim/vterm/state.c)
- [`src/nvim/terminal.c`](../../../neovim/src/nvim/terminal.c)
- [`runtime/doc/terminal.txt`](../../../neovim/runtime/doc/terminal.txt)

## File tree

```text
src/
  nvim/
    vterm/
      state.c
    terminal.c
```

## Big-picture references

- [`runtime/doc/terminal.txt`](../../../neovim/runtime/doc/terminal.txt): documents OSC 52 support and its security rationale.
- [`runtime/doc/dev_arch.txt`](../../../neovim/runtime/doc/dev_arch.txt): explicitly notes that `vterm/` is Nvim-owned, not a Vim-upstream sync surface.
- [`runtime/doc/vim_diff.txt`](../../../neovim/runtime/doc/vim_diff.txt): useful when distinguishing terminal-specific Nvim behavior from legacy editor core behavior.

## Recent fix / history signal

- Adjacent OSC and `TermRequest` fixes show this area has real lifecycle complexity even when the exact bug is not in `osc_selection()`.
- The docs explicitly refuse OSC 52 clipboard reads for security reasons, which is a strong signal that the maintainers already treat this parser as security-sensitive.

## Audit focus

- Review fragment reassembly, especially the `recvpartial` state that carries base64 decode context across callbacks.
- Check invalid-input transitions and whether callbacks can observe partially updated state.
- Verify mask parsing (`clipboard`, `primary`, `secondary`, numbered cut buffers) under malformed prefixes.
- Treat every callback as a possible reentrancy point even though the parser itself is small.

## See Also

- [Terminal TermRequest bridge](13-termrequest.md)
- [TUI CSI and raw input parsing](15-tui-csi-input.md)
