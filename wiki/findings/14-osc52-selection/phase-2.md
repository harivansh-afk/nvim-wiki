```text
src/
  nvim/
    terminal.c
    vterm/
      state.c
    auevents.lua
runtime/
  doc/
    terminal.txt
```

# Phase 2 - OSC 52 dual sink into clipboard and `TermRequest`

## Severity

- Technical severity: medium
- Fleet exposure: medium
- Why: `:terminal` is common, OSC 52 is standard terminal traffic, and the clipboard sink is automatic. The plugin-code sink needs a `TermRequest` handler, but shell-integration plugins and custom terminal workflows do use that event.

## Why this matters on real machines

The interesting part is not the clipboard write by itself. The interesting part is that Neovim tells plugin authors `OSC 52` is consumed internally and not forwarded as a generic terminal request.

If a config or plugin trusts that contract, it may apply looser rules to `TermRequest` than it would apply to clipboard traffic. This bug breaks that separation. A terminal child can make one OSC sequence hit both the internal clipboard path and the editor callback path.

## Technical deep dive

libvterm already recognizes OSC 52 and routes it through `osc_selection()`. That should be the end of it. Instead:

- `osc_selection()` decodes the clipboard payload and hands it to Neovim's selection callback.
- `on_osc()` in `vterm/state.c` still falls through to the generic OSC handler afterwards.
- Neovim's `terminal.c:on_osc()` then rebuilds the raw request and schedules `TermRequest`.

So the same bytes cross two policy boundaries:

1. decoded clipboard data goes to the clipboard provider
2. raw control bytes go to `apply_autocmds_group(EVENT_TERMREQUEST, ...)`

That is exactly the kind of duplicated interpretation bug that turns a documented "handled internally" feature into a plugin-facing attack surface.

## Reproduction on local `nvim v0.12.1`

I ran a terminal child that emitted one OSC 52 sequence while a test clipboard provider and `TermRequest` autocmd were installed.

Observed output:

```json
{"clip":"Hello","termreq":"\u001b]52;;SGVsbG8="}
```

That proves one child-controlled OSC 52 hit both sinks.

## Bottom line

This is not the broadest code-execution bug in the set, but it is one of the more realistic default terminal boundary failures because it crosses from child output into editor automation through a path that the docs claim is isolated.
