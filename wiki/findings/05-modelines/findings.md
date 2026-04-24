# 05-modelines findings

## Broken invariant

`'complete'` is classified as `modelineexpr`, so once an untrusted modeline sets an `F{func}` source, the resulting callback must stay modeline-tainted and execute with the same sandbox policy that Neovim documents for modeline-driven expression options.

Current Neovim loses that policy at the insert-completion sink. The modeline path parses `complete=F{func}` into a stored `Callback`, `option.c` marks the unchecked modeline value insecure, but the later completion sink calls that callback without consulting `was_set_insecurely()` or entering the sandbox. That restores the same capability that direct `'completefunc'` modelines are supposed to forbid: an untrusted file can make a later `<C-N>` / `<C-P>` completion invoke an arbitrary preexisting callback outside the sandbox when `'modelineexpr'` is on.

## Input -> normalization -> policy gate -> sink

1. **Input:** attacker-controlled file text supplies a modeline such as `vim: set complete=FCompletePwn :`.
   - `/home/node/workspace/neovim/src/nvim/buffer.c:3916-3940` (`chk_modeline()`) truncates the `set` fragment and calls `do_set()` with `OPT_MODELINE | OPT_LOCAL` while `secure = 1` and `current_sctx.sc_sid = SID_MODELINE`.
2. **Normalization:** `'complete'` accepts `F{func}` entries and turns them into callback objects.
   - `/home/node/workspace/neovim/src/nvim/options.lua:1469-1470` wires `'complete'` to `did_set_complete()`.
   - `/home/node/workspace/neovim/src/nvim/options.lua:1537-1540` marks `'complete'` as `modelineexpr = true`.
   - `/home/node/workspace/neovim/src/nvim/optionstr.c:863-927` (`did_set_complete()`) validates the option text.
   - `/home/node/workspace/neovim/src/nvim/insexpand.c:3062-3094` (`set_cpt_callbacks()`) parses each `F{func}` entry and stores the normalized `Callback` in `curbuf->b_p_cpt_cb`.
3. **Policy gate:** the modeline layer only checks `'modelineexpr'`, then taints the unchecked option as insecure.
   - `/home/node/workspace/neovim/src/nvim/option.c:1210-1218` allows `'complete'` from a modeline only when `'modelineexpr'` is on.
   - `/home/node/workspace/neovim/src/nvim/option.c:3817-3829` sets `kOptFlagInsecure` for modeline-tainted unchecked options.
   - This matters because direct `'completefunc'` is intentionally forbidden in modelines: `/home/node/workspace/neovim/src/nvim/options.lua:1549-1563`, `/home/node/workspace/neovim/runtime/doc/options.txt:1600-1608`, `/home/node/workspace/neovim/test/old/testdir/test_modeline.vim:209-214`.
4. **Sink:** insert completion recovers and invokes the stored callback without any insecure/modeline sandbox check.
   - `/home/node/workspace/neovim/src/nvim/insexpand.c:4500-4516` (`get_callback_if_cpt_func()`) returns the stored `F{func}` callback.
   - `/home/node/workspace/neovim/src/nvim/insexpand.c:3191-3230` (`expand_by_function()`) reaches `callback_call(cb, 2, args, &rettv)` directly.
   - Unlike other modelineexpr sinks such as `includeexpr`, `indentexpr`, `foldexpr`, `foldtext`, `formatexpr`, and statusline-format evaluation, this path never checks `was_set_insecurely(...)` and never enters the sandbox.

## Minimal reproducer

This PoC is self-contained. The helper function is defined only to make the sink observable in isolation; in a real session the same bug applies to any preexisting completion callback already present through user config, ftplugins, or plugins.

### Files

`/tmp/repro_modeline_complete.vim`

```vim
set modeline modelineexpr
func! CompletePwn(findstart, base)
  call writefile(['pwned:' . a:findstart . ':' . a:base], '/tmp/nvim-modeline-complete-proof')
  if a:findstart
    return 0
  endif
  return ['match']
endfunc
edit /tmp/modeline-complete-attack
call feedkeys("i\<C-N>\<Esc>", 'x')
qall!
```

`/tmp/modeline-complete-attack`

```text
vim: set complete=FCompletePwn :
body
```

### Commands

```bash
cat >/tmp/repro_modeline_complete.vim <<'INNER_REPRO_VIM'
set modeline modelineexpr
func! CompletePwn(findstart, base)
  call writefile(['pwned:' . a:findstart . ':' . a:base], '/tmp/nvim-modeline-complete-proof')
  if a:findstart
    return 0
  endif
  return ['match']
endfunc
edit /tmp/modeline-complete-attack
call feedkeys("i\<C-N>\<Esc>", 'x')
qall!
INNER_REPRO_VIM

cat >/tmp/modeline-complete-attack <<'INNER_ATTACK_FILE'
vim: set complete=FCompletePwn :
body
INNER_ATTACK_FILE

rm -f /tmp/nvim-modeline-complete-proof
nvim --headless -u NONE -i NONE -n -S /tmp/repro_modeline_complete.vim
cat /tmp/nvim-modeline-complete-proof
```

### Expected vs actual

- **Expected:** because the callback source came from a modeline-tainted `modelineexpr` option, later callback execution should be sandboxed. The same helper function under an explicit `:sandbox` call fails with `E48: Not allowed in sandbox` and does not create the proof file.
- **Actual:** `<C-N>` executes `CompletePwn` outside the sandbox and writes `/tmp/nvim-modeline-complete-proof`:

```text
pwned:0:
```

A control run with `'modelineexpr'` left off blocks the modeline at parse time with `E992: Not allowed in a modeline when 'modelineexpr' is off: complete=FCompletePwn`, confirming that the dangerous path is specifically the `modelineexpr`-enabled pipeline.

## Affected files

- `/home/node/workspace/neovim/src/nvim/buffer.c:3916-3940` — modeline parsing plus `do_set()` entry under `secure=1` / `SID_MODELINE`.
- `/home/node/workspace/neovim/src/nvim/options.lua:1469-1470` — `'complete'` callback registration.
- `/home/node/workspace/neovim/src/nvim/options.lua:1537-1540` — `'complete'` marked `modelineexpr`.
- `/home/node/workspace/neovim/src/nvim/optionstr.c:863-927` — `did_set_complete()` validation and callback setup.
- `/home/node/workspace/neovim/src/nvim/option.c:1210-1218` — `modelineexpr` policy gate.
- `/home/node/workspace/neovim/src/nvim/option.c:3817-3829` — modeline-tainted unchecked options marked insecure.
- `/home/node/workspace/neovim/src/nvim/insexpand.c:3062-3094` — `F{func}` text normalized into stored callbacks.
- `/home/node/workspace/neovim/src/nvim/insexpand.c:4500-4516` — stored callback recovered on completion.
- `/home/node/workspace/neovim/src/nvim/insexpand.c:3191-3230` — unsandboxed `callback_call()` sink.
- `/home/node/workspace/neovim/runtime/doc/options.txt:613-618` — documented modeline sandbox contract.
- `/home/node/workspace/neovim/runtime/doc/options.txt:1533-1598` — `'complete'` supports `F{func}` and is gated only by `'modelineexpr'`.
- `/home/node/workspace/neovim/runtime/doc/options.txt:1600-1608` — direct `'completefunc'` is disallowed in modelines.
- `/home/node/workspace/neovim/test/old/testdir/test_modeline.vim:209-214` — regression coverage exists for blocking direct `'completefunc'`, but not for the indirect `'complete'` callback path.

## Impact / severity

This is a **moderate-severity modeline-to-callback-execution bypass**.

Why it matters:

- **Security boundary violation:** Neovim explicitly bans direct `'completefunc'` modelines, but a modeline can still select an equivalent callback sink indirectly through `'complete'`.
- **Unsandboxed execution:** the later callback runs outside the sandbox even though the option value was marked modeline-tainted.
- **Useful primitives:** completion callbacks can run Vimscript/Lua logic with side effects such as file writes, command execution, job launch, RPC, or network/plugin activity depending on the configured callback body.

Why it is not higher severity:

- **Opt-in reachability:** the user must have `'modelineexpr'` enabled; its default is off.
- **Second user action required:** the callback fires when insert completion is invoked (`<C-N>`, `<C-P>`, or related completion flows), not at file-open time.

Even with those constraints, this is still a real exploit shape rather than a weird corner case: it is a clear policy/sink drift inside the documented modeline security model, and it recreates a callback-execution capability that the direct modeline denylist is already trying to block.
