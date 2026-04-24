```text
src/
  nvim/
    spellfile.c
    memory.c
runtime/
  doc/
    spell.txt
```

# Phase 2 - `COMPOUNDFLAG` raw spelling overflow in `:mkspell`

## Severity

- Technical severity: medium
- Fleet exposure: low
- Why: the sink is a real stack overflow in release builds, but the victim must run `:mkspell` on attacker-controlled spell sources.

## Why this is niche but real

This is not a normal file-open bug. Most users will never feed untrusted `.aff` data into `:mkspell`. But the bug is still worth tracking because the vulnerable path is a documented external input boundary, and the failure mode is memory corruption rather than a clean parse error.

It is most relevant to people who build spell resources, package language files, or process third-party dictionaries in tooling or CI.

## Technical deep dive

The interesting part is not the numeric parse itself. The interesting part is that Neovim keeps both the parsed numeric value and the attacker-chosen raw spelling alive.

- `FLAG num` switches the parser into numeric-flag mode.
- `COMPOUNDFLAG` stores the raw token in `compflags` without an `AH_KEY_LEN` check.
- `process_compflags()` later parses the flag value with `get_affitem()`, so the token is considered valid.
- but the same function then uses the original raw spelling as a hash key and copies it with `xmemcpyz()` into `char key[AH_KEY_LEN]`.

That is why a spelling like `00000000000000001` is dangerous even though it numerically means just `1`. Validation happens on the integer interpretation, while the overflow happens on the preserved original text.

## Reproduction on local `nvim v0.12.1`

I reran the direct `:mkspell` proof with a crafted `.aff` containing `COMPOUNDFLAG 00000000000000001`.

Observed result:

```text
rc=134
Reading affix file /tmp/nvim-phase2b/poc.aff...*** buffer overflow detected ***: terminated
```

## Bottom line

This is a legitimate stack overflow on a documented external parser surface, but it is not one of the most broadly deployable user-machine compromise paths because `:mkspell` is a specialist workflow.
