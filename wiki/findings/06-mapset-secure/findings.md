# modeline `ft=` / `syn=` selectors reopen secure-mode mapping sinks

## Broken invariant

A modeline-controlled selector may validate the selector token itself, but it must **not** clear or bypass `secure` before running the selector's handlers.

In current Neovim, `'filetype'` / `'syntax'` modelines are treated as "safe" once `valid_filetype()` accepts the name. That is the wrong invariant. The selector string is syntactically safe, but the **FileType/Syntax autocmd bodies are arbitrary Ex/Lua code** and can still mutate mappings.

## Input -> normalization -> policy gate -> sink

### Primary chain (`filetype`)

1. **Input**: attacker-controlled file content with a modeline such as `vim: set ft=pwn :`.
2. **Normalization**: `did_set_filetype_or_syntax()` accepts `pwn` via `valid_filetype()` and marks the value as checked (`os_value_checked = true`).
3. **Policy gate drift**:
   - modelines enter `secure = 1` in `do_modelines()`;
   - `option.c` still triggers `FileType` autocmds from a modeline when the value changed;
   - `do_filetype_autocmd()` then explicitly resets `secure = 0` because the *selector value* was checked.
4. **Sink**: the selected `FileType` handler runs arbitrary commands such as `nnoremap <buffer> ...`, which reaches `ex_map()` / `do_exmap()` / `buf_do_map()` and installs a mapping.

### Sibling chain (`syntax`)

`vim: set syn=pwn :` follows the same selector-validation path, then reaches `Syntax` autocmd handlers. `do_syntax_autocmd()` does not clear `secure`, but that still does **not** save the boundary:

- Ex `:map` handlers still execute in secure mode because `ex_map()` only echoes them;
- Lua autocmd callbacks run via `nlua_call_ref()` with no `check_secure()`.

So both selector families still reach mapping mutation sinks from modelines.

## Minimal reproducer

Verified with the available `/home/node/.nix-profile/bin/nvim` (`NVIM v0.12.1`). The checked-out source at `/home/node/workspace/neovim` still contains the same trigger/reset/callback path cited below.

### File contents

`/tmp/pwn-modeline.txt`

```text
hello
# vim: set ft=pwn :
```

### Commands

```bash
cat >/tmp/pwn-modeline.txt <<'EOF'
hello
# vim: set ft=pwn :
EOF

nvim -u NONE -i NONE -n --headless \
  --cmd "set nomore modeline modelines=1" \
  --cmd "autocmd FileType pwn nnoremap <buffer> gP :let b:pwn=1<CR>" \
  /tmp/pwn-modeline.txt \
  "+call writefile([string(maparg('gP','n',0,1))], '/tmp/pwn-map.txt')" \
  "+qa!"

cat /tmp/pwn-map.txt
```

### Expected vs actual

- **Expected**: opening an untrusted file should not let a modeline reach a mapping sink; `/tmp/pwn-map.txt` should contain `''` (or Neovim should raise a secure/sandbox error before the mapping is created).
- **Actual**: `/tmp/pwn-map.txt` contains the created buffer-local mapping dictionary, e.g.:

```text
{'lhs': 'gP', 'mode': 'n', 'expr': 0, 'sid': -2, 'lnum': 0, 'noremap': 1, 'nowait': 0, 'rhs': ':let b:pwn=1<CR>', 'lhsraw': 'gP', 'abbr': 0, 'script': 0, 'replace_keycodes': 0, 'mode_bits': 1, 'silent': 0, 'buffer': 1, 'scriptversion': 1}
```

### Variant (`syntax`)

This also works with `# vim: set syn=pwn :` plus either:

- `autocmd Syntax pwn nnoremap <buffer> gS :let b:syn_ex=1<CR>` (echoed, but still executed), or
- a Lua `Syntax` callback that calls `vim.api.nvim_buf_set_keymap()`.

## Affected files

- `src/nvim/buffer.c:3932-3942` — modeline processing enters `secure = 1` before `do_set()`.
- `src/nvim/optionstr.c:1231-1243` — `did_set_filetype_or_syntax()` validates the selector with `valid_filetype()` and marks it `os_value_checked = true`.
- `src/nvim/options.lua:3162-3170` — `'filetype'` is documented to trigger `FileType` and be useful in modelines.
- `src/nvim/options.lua:9132-9155` — `'syntax'` is documented to trigger `Syntax` and be useful in modelines.
- `src/nvim/option.c:3776-3784` — `FileType`/`Syntax` autocmds are triggered after the option is set; the `FileType` branch even has a stale comment claiming modelines are skipped.
- `src/nvim/autocmd.c:2741-2755` — `do_filetype_autocmd()` resets `secure = 0` before running `FileType` handlers.
- `src/nvim/option.c:2940-2949` — `do_syntax_autocmd()` runs `Syntax` handlers from the selected syntax value.
- `src/nvim/mapping.c:2700-2708` — `ex_map()` in secure mode only echoes the mapping command, then still executes it.
- `src/nvim/autocmd.c:2132-2166` and `src/nvim/lua/executor.c:1831-1835` — Lua autocmd callbacks are dispatched via `nlua_call_ref()` without a `check_secure()` gate.
- `src/nvim/mapping.c:561-903` — `do_exmap()` / `buf_do_map()` ultimately install the mapping.

## Why this matters

This is the same secure-boundary class as the recent `mapset()` fix, just reopened through selector side effects instead of the function itself.

- It does **not** require `'modelineexpr'`.
- `'modeline'` is on by default for non-root users (`src/nvim/options.lua:5842-5857`).
- Neovim's own docs and tests explicitly encourage `ft=` / `syn=` in modelines.
- Real configs and plugins commonly attach `FileType` / `Syntax` handlers that set buffer-local mappings, commands, keymaps, or richer Lua behavior.

So an attacker-controlled file can choose which existing trusted handler graph runs on open. The minimum verified impact is unauthorized mapping creation from file content. In real installations this can easily expand to broader callback/plugin execution, depending on what the victim has registered on `FileType` / `Syntax`.

## Coverage gap

Current tests actually lock in the risky behavior instead of defending against it:

- `test/old/testdir/test_modeline.vim:35-49` asserts that `vim: set ft=c :` fires ftplugin logic (`b:did_ftplugin`, `&ofu`).
- `test/old/testdir/test_modeline.vim:51-62` asserts that `vim: set syn=c :` loads syntax.

There is no regression test that says modeline-triggered `FileType` / `Syntax` handlers must **not** reach mapping sinks.
