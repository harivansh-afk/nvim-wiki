# OSC 52 is handled twice: clipboard write plus unexpected `TermRequest`

## Broken invariant

Once Neovim recognizes `OSC 52` as a clipboard-selection sequence, it must be consumed by the clipboard path and must **not** also reach the generic `TermRequest` bridge.

Today that invariant is broken. `src/nvim/vterm/state.c` handles `OSC 52` via `osc_selection()`, but `on_osc()` still unconditionally invokes the fallback OSC handler afterwards (`src/nvim/vterm/state.c:1977-1986`). Neovim's fallback handler in `terminal.c` then reserializes the same raw sequence and schedules a `TermRequest` autocmd (`src/nvim/terminal.c:349-383`, `src/nvim/terminal.c:305-317`, `src/nvim/terminal.c:236-282`).

That contradicts both the docs and the event metadata:

- `runtime/doc/terminal.txt:200-201` says `OSC 52` from `:terminal` is handled directly by Nvim and is **not** forwarded to plugins.
- `src/nvim/auevents.lua:124` describes `TermRequest` as firing after an **unhandled** OSC sequence.

## Input -> normalization -> policy gate -> sink

1. **Input**: a program running inside `:terminal` emits `ESC ] 52 ;; SGVsbG8= ESC \`.
2. **Normalization**:
   - `terminal_receive()` passes child-process bytes into libvterm with `vterm_input_write()` (`src/nvim/terminal.c:1381-1402`).
   - libvterm routes OSC command `52` into `osc_selection()` (`src/nvim/vterm/state.c:1959-1980`).
   - `osc_selection()` parses the selector, defaults an empty selector, base64-decodes fragments, and emits decoded chunks through the selection callback (`src/nvim/vterm/state.c:1796-1957`).
   - Neovim's selection callback appends the decoded bytes and schedules a clipboard-provider write on the main loop (`src/nvim/terminal.c:1844-1856`), which lands in `eval_call_provider("clipboard", "set", ...)` (`src/nvim/terminal.c:1813-1842`).
3. **Broken policy gate**:
   - Even after handling command `52`, `vterm/state.c:on_osc()` still calls the fallback OSC handler (`src/nvim/vterm/state.c:1984-1986`) instead of treating the sequence as consumed.
   - Neovim's fallback `on_osc()` in `terminal.c` accepts all OSC commands whenever a `TermRequest` autocmd exists, not just genuinely unhandled ones (`src/nvim/terminal.c:358-383`).
4. **Sinks**:
   - **Clipboard sink**: `term_clipboard_set()` -> `eval_call_provider("clipboard", "set", ...)` (`src/nvim/terminal.c:1813-1842`).
   - **Plugin-code sink**: `schedule_termrequest()` / `emit_termrequest()` -> `apply_autocmds_group(EVENT_TERMREQUEST, ...)` (`src/nvim/terminal.c:305-317`, `src/nvim/terminal.c:236-282`).

So one attacker-controlled `OSC 52` sequence crosses two boundaries at once: it writes the clipboard **and** reaches editor-side autocmd/plugin code that the docs say should never see it.

## Why this violates the documented contract

Neovim documents two separate policies for this surface:

- `runtime/doc/terminal.txt:195-198` keeps clipboard reads disabled for security.
- `runtime/doc/terminal.txt:200-201` says clipboard-write `OSC 52` is handled internally and does not emit `TermRequest`.

The implementation only enforces the first policy. It disables direct clipboard-query callbacks by setting `.query = NULL` (`src/nvim/terminal.c:228-231`), but it forgets to enforce the second policy because the generic fallback bridge still runs after `OSC 52` is already recognized and handled.

## Minimal reproducer

### Reproducer file

Save this as `/tmp/osc52-dual-sink.lua`:

```lua
vim.g.termreq = 'unset'
vim.g.clip = 'unset'

vim.api.nvim_create_autocmd('TermRequest', {
  callback = function()
    vim.g.termreq = vim.v.termrequest
  end,
})

vim.g.clipboard = {
  name = 'Test',
  copy = {
    ['+'] = function(lines)
      vim.g.clip = table.concat(lines, '\n')
    end,
    ['*'] = function() end,
  },
  paste = {
    ['+'] = function()
      return { '' }, 'v'
    end,
    ['*'] = function()
      return { '' }, 'v'
    end,
  },
}

vim.cmd([[terminal sh -c "sleep 0.1; printf '\033]52;;SGVsbG8=\033\\'; sleep 0.1"]])
vim.wait(2000)

local out = io.open('/tmp/osc52-result.json', 'w')
out:write(vim.json.encode({
  termreq = vim.g.termreq,
  clip = vim.g.clip,
}), '\n')
out:close()

vim.cmd('qa!')
```

### Commands

```bash
cat > /tmp/osc52-dual-sink.lua <<'EOF'
vim.g.termreq = 'unset'
vim.g.clip = 'unset'

vim.api.nvim_create_autocmd('TermRequest', {
  callback = function()
    vim.g.termreq = vim.v.termrequest
  end,
})

vim.g.clipboard = {
  name = 'Test',
  copy = {
    ['+'] = function(lines)
      vim.g.clip = table.concat(lines, '\n')
    end,
    ['*'] = function() end,
  },
  paste = {
    ['+'] = function()
      return { '' }, 'v'
    end,
    ['*'] = function()
      return { '' }, 'v'
    end,
  },
}

vim.cmd([[terminal sh -c "sleep 0.1; printf '\033]52;;SGVsbG8=\033\\'; sleep 0.1"]])
vim.wait(2000)

local out = io.open('/tmp/osc52-result.json', 'w')
out:write(vim.json.encode({
  termreq = vim.g.termreq,
  clip = vim.g.clip,
}), '\n')
out:close()

vim.cmd('qa!')
EOF

script -q -c "/home/node/.nix-profile/bin/nvim --clean -u NONE -S /tmp/osc52-dual-sink.lua" \
  /tmp/osc52-typescript >/dev/null 2>&1

cat /tmp/osc52-result.json
```

### Expected

Because `runtime/doc/terminal.txt:200-201` says `OSC 52` does **not** emit `TermRequest`, the terminal child should only update the clipboard provider:

```json
{"termreq":"unset","clip":"Hello"}
```

### Actual

On this box, the same single `OSC 52` sequence hits both sinks:

```json
{"termreq":"\u001b]52;;SGVsbG8=","clip":"Hello"}
```

So Neovim both:

- writes decoded text to the clipboard provider (`"clip":"Hello"`), and
- exposes the raw `OSC 52` request to `TermRequest` handlers (`"termreq":"\u001b]52;;SGVsbG8="`).

## Affected files

- `src/nvim/vterm/state.c:1796-1957` — `osc_selection()` parses selector and base64 payload, then calls the selection callback.
- `src/nvim/vterm/state.c:1977-1986` — `on_osc()` handles command `52` but still calls the fallback handler afterwards.
- `src/nvim/terminal.c:1844-1856` — `term_selection_set()` accumulates decoded clipboard data and schedules the clipboard write.
- `src/nvim/terminal.c:1813-1842` — `term_clipboard_set()` maps the mask to a register and calls the clipboard provider.
- `src/nvim/terminal.c:349-383` — fallback `on_osc()` reserializes the raw sequence for `TermRequest`.
- `src/nvim/terminal.c:305-317` — `schedule_termrequest()` queues the event.
- `src/nvim/terminal.c:236-282` — `emit_termrequest()` sets `v:termrequest` and runs `apply_autocmds_group(EVENT_TERMREQUEST, ...)`.
- `runtime/doc/terminal.txt:188-201` — documented `OSC 52` behavior and the promise that it is not forwarded to plugins.
- `src/nvim/auevents.lua:124` — `TermRequest` metadata says it is for unhandled OSC sequences.

## Impact / severity

This is a **policy-boundary violation** in a default `:terminal` input surface.

The clipboard write is documented, but the extra `TermRequest` delivery is not. That means an untrusted terminal program can reach editor-side Lua/Vimscript autocmd handlers through a path Neovim explicitly promises is isolated from plugins.

That matters because `TermRequest` is a code-execution bridge, not just a logging hook: its sink is `apply_autocmds_group(...)`, and Neovim's own docs encourage users and plugins to attach behavior to it for OSC-driven features such as directory tracking and shell integration. Any config or plugin that trusts the docs and assumes `OSC 52` will never arrive there is wrong.

I would rate this **medium severity**:

- it is not direct code execution in stock Neovim by itself;
- but it is a clean, reproducible boundary break from terminal-child bytes into high-privilege editor callbacks;
- and it defeats a specific security/documentation invariant on a default-on core surface.
