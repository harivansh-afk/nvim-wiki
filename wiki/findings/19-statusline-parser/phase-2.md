```text
src/
  nvim/
    statusline.c
    grid.c
    optionstr.c
    api/
      vim.c
runtime/
  doc/
    options.txt
```

# Phase 2 - grouped statusline right-alignment overflow

## Severity

- Technical severity: high
- Fleet exposure: low-medium
- Why: the bug is real C memory corruption in a ubiquitous renderer, but the attacker still needs control of a statusline format or `nvim_eval_statusline()` input. That is common in plugin and RPC ecosystems, not in plain file-open flows.

## Why this is severe but not the broadest user-machine bug

This finding is technically one of the nastiest in the repo. I reproduced both a stack smash and a heap corruption path. But its real-world spread depends on whether untrusted input can reach a statusline format sink on the victim machine.

So the correct read is:

- very high engine severity
- weaker default delivery than modeline, ShaDa, or ordinary terminal surfaces

## Technical deep dive

The vulnerable branch in `build_stl_str_hl()` mixes two units that must stay separate:

1. display cells missing from the group
2. bytes needed when the fill glyph is multibyte

For right-aligned groups it does:

- `group_len = (min_group_width - group_len) * schar_len(fillchar)`
- `memmove()` by that byte count
- then uses the same `group_len` as the fill loop bound

But `schar_get_adv()` copies the full multibyte glyph each iteration. So the code shifts by bytes, then fills by "bytes interpreted as number of glyph copies", which overwrites the moved content and can run off the destination buffer.

The bug hits both:

- `win_redr_custom()` with stack `char buf[MAXPATHL]`
- `nvim_eval_statusline()` with an arena buffer

## Reproduction on local `nvim v0.12.1`

I reproduced both sinks.

Observed results:

```text
statusline_rc=134
*** stack smashing detected ***: terminated

api_rc=134
double free or corruption (!prev)
```

## Bottom line

This is a high-severity memory corruption bug, but not one of the very broadest "many user machines today" findings unless you can show an actual path from untrusted workspace data, plugin input, or RPC input into a statusline format string.
