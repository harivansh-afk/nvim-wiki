```text
src/
  nvim/
    buffer.c
    option.c
    optionstr.c
    autocmd.c
    mapping.c
```

# Phase 2 - modeline `ft=` / `syn=` selectors

## Severity

- Technical severity: medium-high
- Fleet exposure: high
- Why: `'modeline'` is on by default for normal users, `ft=` and `syn=` are documented modeline features, and real configs are dense with `FileType` and `Syntax` handlers.

## Why this lands on many real machines

This finding is dangerous because it does not need a niche mode or a synthetic sink. It lets file content choose an existing trusted handler graph that the victim already installed. On a stock build the effect may be small, but on a real setup `FileType` and `Syntax` autocmds often set mappings, commands, callbacks, buffer options, and Lua helpers.

The important shift is this:

1. the modeline only needs to pick a valid selector name
2. Neovim treats that selector as safe
3. the selected handler bodies are not safe at all

That makes this much more deployable than bugs that require `--remote-ui`, `:mkspell`, or a crafted state-file path.

## Technical deep dive

`chk_modeline()` enters the option layer with `secure = 1`, but the policy only follows the selector string, not the side effects of the selected handler.

- `did_set_filetype_or_syntax()` validates the selector with `valid_filetype()` and marks the value checked.
- `option.c` still triggers `FileType` or `Syntax` autocmds after the option is set.
- `do_filetype_autocmd()` then drops `secure` before the handler graph runs.
- Even the `Syntax` branch is not safe, because mapping and Lua callback sinks are still reachable from the autocmd body.

So the real bug is not "bad filetype validation". The bug is that Neovim treats "the selector token is syntactically valid" as if it meant "everything the selected handler does is safe under modeline semantics".

## Reproduction on local `nvim v0.12.1`

I reproduced the `FileType` path with a one-line modeline and a buffer-local mapping sink.

Observed output:

```text
{'lhs': 'gP', 'mode': 'n', 'expr': 0, 'sid': -2, 'lnum': 0, 'noremap': 1, 'nowait': 0, 'rhs': ':let b:pwn=1<CR>', 'lhsraw': 'gP', 'abbr': 0, 'script': 0, 'replace_keycodes': 0, 'mode_bits': 1, 'silent': 0, 'buffer': 1, 'scriptversion': 1}
```

That confirms file content created a buffer-local mapping through a modeline-selected handler path.

## Bottom line

This is one of the best "broad user machine" findings in the set. The open-file trigger is ordinary, the feature is default-on, and the exploit surface scales with the victim's plugin graph.
