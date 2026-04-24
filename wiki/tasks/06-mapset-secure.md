# mapset secure-mode gate

## Overview

This is a narrow mixed-policy boundary rather than a broad subsystem. [`src/nvim/mapping.c`](../../../neovim/src/nvim/mapping.c) exposes `f_mapset()`, and [`src/nvim/ex_cmds.c`](../../../neovim/src/nvim/ex_cmds.c) provides `check_secure()`, the gate used to reject operations in secure or sandboxed contexts. It ranks highly because the recent modeline bypass fix landed exactly in this neighborhood: it is a concentrated example of how one missing or misapplied gate can break the whole trust model.

## Relevant options and knobs

- `'modeline'`: relevant because modelines are one path into secure-mode restrictions.
- `'modelineexpr'`: relevant because expression-capable settings often determine how much code can be reached from file content.
- `'exrc'`: another policy-adjacent surface that depends on the same broad secure/sandbox concept.
- No single editor option controls `mapset()` itself; the important “knob” is whether the operation is reached from a restricted context.

## Relevant files

- [`src/nvim/mapping.c`](../../../neovim/src/nvim/mapping.c)
- [`src/nvim/ex_cmds.c`](../../../neovim/src/nvim/ex_cmds.c)
- [`src/nvim/options.lua`](../../../neovim/src/nvim/options.lua)

## File tree

```text
src/
  nvim/
    mapping.c
    ex_cmds.c
    options.lua
```

## Big-picture references

- [`runtime/doc/message.txt`](../../../neovim/runtime/doc/message.txt): covers the classic secure-mode error messages exposed to users.
- [`runtime/doc/options.txt`](../../../neovim/runtime/doc/options.txt): explains modeline and sandbox restrictions for option-setting behavior.
- [`runtime/doc/editing.txt`](../../../neovim/runtime/doc/editing.txt): complements the trust/secure picture with the newer trusted-files model.

## Recent fix / history signal

- `c7604323e3` / `c084ab9f57`: modeline security bypass fix (#38657).
- The fact that a specific function-level gate mattered more than broad subsystem behavior is exactly why this region deserves a standalone ticket.

## Audit focus

- Enumerate every operation that should be blocked in secure mode and compare them against actual `check_secure()` callsites.
- Review mapping creation paths that accept callbacks or Lua refs; policy bugs become more serious when the payload is itself executable code.
- Check for generated-policy drift between `options.lua` and handwritten checks in C.
- When auditing future changes, treat this as a regression hotspot, not a one-off fixed bug.

## See Also

- [Modeline execution pipeline](05-modelines.md)
- [Project-local exrc loader](07-exrc-loader.md)
- [Trust database and :trust bridge](08-trust-db.md)
