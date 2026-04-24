```text
src/
  nvim/
    shada.c
    eval.c
    eval/
      vars.c
runtime/
  doc/
    options.txt
```

# Phase 2 - ShaDa variable name overflow on writeback

## Severity

- Technical severity: high
- Fleet exposure: medium-high
- Why: `'shada'` is read on startup and written on exit by default. This is part of the normal interactive lifecycle for many users, not a niche feature.

## Why this one is broadly deployed

Many Neovim findings only matter if the victim opts into a special mode. This one sits in the default state-file path that most interactive users already have enabled.

The delivery constraint is still real: the attacker needs control of the ShaDa file or the path Neovim reads. But once that boundary is crossed, the rest of the exploit is just normal start and normal exit. No extra command or rare mode is needed.

## Technical deep dive

This is a classic representation-drift bug across phases:

- `shada_read_next_item()` accepts the variable name as an arbitrary MessagePack string.
- `var_set_global()` and `valid_varname()` validate the identifier syntax, but not the length needed by later serializer scratch buffers.
- uppercase names are classified for ShaDa persistence by default.
- on writeback, `shada_pack_entry()` builds `"variable g:" + name + "\0"` into `char vardesc[256]` with `memcpy(..., varname.size + 1)`.

So the read path and the in-memory variable layer both say the name is fine, but the write path silently assumes the same name still fits a 256-byte local description buffer.

## Reproduction on local `nvim v0.12.1`

I verified both halves of the chain:

- control run with writeback disabled printed `0`, which shows the 245-byte uppercase variable was accepted and restored into `g:`
- normal exit crashed during writeback

Observed crash:

```text
rc=134
*** buffer overflow detected ***: terminated
```

## Bottom line

This is a strong "default path" memory corruption bug. It is not as easy to deliver as a file-open modeline issue, but it lives on the normal startup and shutdown path of many real installs.
