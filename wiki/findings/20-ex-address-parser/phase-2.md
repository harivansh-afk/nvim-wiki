```text
src/
  nvim/
    ex_docmd.c
    charset.c
    pos_defs.h
runtime/
  doc/
    cmdline.txt
```

# Phase 2 - absolute Ex address truncation and wrap

## Severity

- Technical severity: medium
- Fleet exposure: medium
- Why: the Ex parser is core and widely deployed, but the bug is an integrity failure rather than a memory corruption bug. It matters most when wrappers, plugins, or RPC bridges trust Neovim to reject impossible line numbers.

## Why this matters despite not being memory corruption

This bug sits at a front-door parser in front of a huge command surface. If a higher layer forwards attacker-controlled Ex text and assumes Neovim will reject absurd addresses, that assumption is false.

The parser does not fail closed. It narrows first, validates later, and executes on the narrowed line.

## Technical deep dive

The core mistake is unit narrowing before policy:

- `get_address()` takes the absolute-number path and does `lnum = (linenr_T)getdigits(...)`
- `getdigits()` can already collapse huge values to a default on parse overflow
- values that fit `intmax_t` but not `linenr_T` can still wrap on the cast
- `invalid_range()` validates the already-truncated `linenr_T`
- `correct_range()` then upgrades `0` to line `1` for ordinary commands

So the command layer is not operating on the user's actual decimal input. It is operating on a parser artifact produced by overflow and truncation.

## Reproduction on local `nvim v0.12.1`

I reproduced all three parts:

```text
nvim_parse_cmd('999...delete') -> "range":[0]
999...delete on alpha/beta/gamma -> beta\ngamma
4294967298delete on alpha/beta/gamma -> alpha\ngamma
```

That means the huge value was not rejected. It became either `0 -> 1` or a wrapped positive line.

## Bottom line

This is not one of the strongest compromise bugs by itself. It is a core-parser integrity bug. Its importance comes from composition: any helper that passes untrusted Ex addresses downstream and relies on Neovim to reject nonsense can be steered into acting on the wrong line.
