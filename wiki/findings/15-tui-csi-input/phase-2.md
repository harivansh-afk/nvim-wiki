```text
src/
  nvim/
    tui/
      input.c
      termkey/
        driver-csi.c
```

# Phase 2 - CSI subparameter stack overwrite

## Severity

- Technical severity: medium-high
- Fleet exposure: medium
- Why: this is default-on TUI input parsing on almost every terminal-driven Neovim session, but the attacker must control host terminal input bytes, not just file contents or terminal child output.

## Why this one sits in the middle

This finding matters because it is in the front-door TUI input path, not in an opt-in subsystem. Every terminal user passes through this code. But the delivery condition is unusual: the hostile bytes come from the outer terminal layer, so the attacker typically needs a malicious terminal emulator, multiplexer, transport proxy, or another component that can spoof input reports.

That makes it broader than `:mkspell` or `--remote-ui`, but less directly exploitable than open-file modeline or state-file bugs.

## Technical deep dive

The bug is a clean parser-capacity mismatch:

- the CSI handlers for SS3, `~`, and CSI-u allocate a single `int subparam` and set `nsubparams = 1`
- `parse_csi()` preserves `:` inside a parameter slice, so `1:2:3` is handed downstream intact
- `termkey_interpret_csi_param()` loops while `length <= capacity`
- after the second `:`, the epilogue writes `subparams[length - 1]`, which becomes `subparams[1]`

So the parser does not need a giant input or a malformed outer frame. It only needs an extra subparameter in an otherwise recognized key sequence.

## Reproduction on local toolchain

I reproduced the exact helper body and caller shape with an AddressSanitizer harness that matches the real Neovim call sites.

Observed result:

```text
ERROR: AddressSanitizer: stack-buffer-overflow
WRITE of size 4
... in termkey_interpret_csi_param
```

I am intentionally treating the helper-level overwrite as the core proof here, because the end-to-end effect in a release TUI build depends on stack layout after a one-slot overwrite. The underlying write itself is not ambiguous.

## Bottom line

This is a real memory-safety bug on a default TUI path, but the compromise story depends on control of host input rather than ordinary workspace data. It is severe enough to fix quickly, but not one of the easiest broad delivery paths.
