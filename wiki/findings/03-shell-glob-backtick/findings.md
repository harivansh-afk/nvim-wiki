# Modeline-controlled `'path'` entries reach shell-backed glob expansion

## Broken invariant

`'path'` is documented as a list of directory names used for file lookup (`gf`, `:find`, etc.), so a `'path'` entry — even when it comes from untrusted file content via modeline — is supposed to remain passive path-selection data.

> a `'path'` directory entry must not be reinterpreted as active shell / backtick syntax when later code glob-expands against it.

Current Neovim violates that invariant. A buffer-local `'path'` value seeded from modeline is later passed into the generic `globpath()` / wildcard-expansion stack without any backtick ban. Once a caller globs against `&path`, the raw path item is concatenated with the requested filename pattern, `SPECIAL_WILDCHAR` logic treats the embedded backticks as executable syntax, and the shell runs the payload.

This is especially notable because nearby sibling surfaces already encode the expected policy:

- `'dictionary'` and `'thesaurus'` explicitly say “Backticks cannot be used in this option for security reasons”.
- tag filenames were hardened in C with an explicit `vim_strchr(fname, '`') == NULL` check before expansion.

`'path'` lacks the equivalent guard.

## Input -> normalization -> policy gate -> sink

1. **Input:** attacker-controlled file content sets a buffer-local `'path'` value via modeline.
   - `runtime/doc/options.txt:4451-4470` documents that `'modeline'` is on by default (except root) and checked on file open.
   - `runtime/doc/options.txt:4806-4855` documents `'path'` as a list of directories used for file lookup.
   - `src/nvim/options.lua:6555-6562` defines `'path'` metadata without `secure = true`.

2. **Normalization:** a later caller passes `&path` into `globpath()`.
   - `src/nvim/eval/fs.c:878-910` (`f_globpath()`) forwards the provided path string directly into `globpath()`.
   - `src/nvim/cmdexpand.c:3596-3633` (`globpath()`) splits the comma-separated path with `copy_option_part()`, concatenates each raw path item with the requested filename pattern, and hands the result to `ExpandFromContext()`.
   - There is no backtick stripping or rejection in this step.

3. **Policy gate:** there is no `'path'`-specific backtick ban before wildcard expansion.
   - `runtime/doc/options.txt:2150-2162` and `6889-6896` explicitly ban backticks for `'dictionary'` and `'thesaurus'`.
   - `src/nvim/tag.c:3078-3085` explicitly refuses backticks before expanding tag filenames.
   - `runtime/doc/options.txt:4806-4855` and `src/nvim/options.lua:6555-6562` provide no equivalent restriction for `'path'`.

4. **Sink:** the concatenated `path-item + filename-pattern` string reaches the shell-backed wildcard engine.
   - `src/nvim/cmdexpand.c:3633` calls `ExpandFromContext()` on the concatenated string.
   - `src/nvim/path.c:1235-1251` (`has_special_wildchar()`) treats backticks as special wildcard characters.
   - `src/nvim/path.c:1297-1305` routes such patterns to `os_expand_wildcards()`.
   - `src/nvim/os/shell.c:295-327` toggles `intick` on backticks and deliberately leaves bytes inside backticks active instead of escaping them.
   - `src/nvim/os/shell.c:353-355` finally executes the constructed shell command with `call_shell()`.

A second built-in path to the same sink exists in completion plumbing:

- `src/nvim/cmdexpand.c:2711-2722` maps `EXPAND_FILES_IN_PATH` / `file_in_path` completion to `expand_wildcards_eval(..., EW_PATH)`.
- `src/nvim/path.c:1172-1202` (`expand_in_path()`) routes that through the same `globpath()` helper.

So this is not just an explicit `globpath(&path, ...)` footgun; the same modeline-seeded option value also reaches built-in `file_in_path` completion logic.

## Why this breaks the documented contract

- `runtime/doc/options.txt:4809-4845` describes `'path'` entries as **directories** to search, not as commands or active mini-programs.
- `runtime/doc/options.txt:4451-4470` makes modelines a default-on project-content input channel.
- `runtime/doc/options.txt:2150-2162` and `6889-6896` show the expected defensive policy for comparable file-list options: backticks are forbidden because they would become execution syntax later.
- `src/nvim/tag.c:3078-3085` shows the same invariant restored in code for tag filename expansion.

So the intended model is “path-like option entries are data”. The bug is that `'path'` entries are later reinterpreted by the shell wildcard engine as executable backtick syntax.

## Minimal reproducer

This reproduces on stock Neovim with `--clean`; no user config is needed.

On this box, `nvim --clean` uses `shell=/usr/bin/zsh`, so the proof payload below uses a zsh-compatible no-space backtick command.

### Files

`/tmp/nvimwiki-path-victim.c`

```c
int main(void) { return 0; }
// vim: set path=`cat<<<PWN>/tmp/nvimwiki-path-pwn;cat<<</tmp/nvimwiki-pathdir`:
```

`/tmp/nvimwiki-pathdir/target.txt`

```text
target
```

### Commands

```bash
rm -rf /tmp/nvimwiki-pathdir /tmp/nvimwiki-path-pwn /tmp/nvimwiki-path-victim.c \
  /tmp/nvimwiki-globpath-modeline-marker.txt
mkdir -p /tmp/nvimwiki-pathdir
printf 'target\n' >/tmp/nvimwiki-pathdir/target.txt
cat > /tmp/nvimwiki-path-victim.c <<'EOF'
int main(void) { return 0; }
// vim: set path=`cat<<<PWN>/tmp/nvimwiki-path-pwn;cat<<</tmp/nvimwiki-pathdir`:
EOF
rm -f /tmp/nvimwiki-path-pwn

nvim --clean --headless /tmp/nvimwiki-path-victim.c \
  "+call writefile([string(globpath(&path, 'target.txt', 0, 1))], '/tmp/nvimwiki-globpath-modeline-marker.txt')" \
  '+qall!'

cat /tmp/nvimwiki-path-pwn
cat /tmp/nvimwiki-globpath-modeline-marker.txt
```

### Expected vs actual

- **Expected:** the modeline-provided `'path'` entry should be treated as a directory literal or rejected as invalid path data. Calling `globpath(&path, 'target.txt', ...)` should not execute shell syntax embedded in the option value.
- **Actual:** the backtick payload in `'path'` is executed. The proof file contains:

```text
PWN
```

and `globpath()` returns the attacker-selected directory match:

```text
['/tmp/nvimwiki-pathdir/target.txt']
```

That is, a path-list option entry crossed the boundary from passive lookup data into active shell execution.

## Affected files

- `runtime/doc/options.txt:4451-4470` — modelines are default-on and processed from file content.
- `runtime/doc/options.txt:4806-4855` — `'path'` is documented as directory-list lookup data.
- `runtime/doc/options.txt:2150-2162` — `'dictionary'` explicitly bans backticks for security.
- `runtime/doc/options.txt:6889-6896` — `'thesaurus'` explicitly bans backticks for security.
- `src/nvim/options.lua:6555-6562` — `'path'` metadata lacks `secure = true`.
- `src/nvim/eval/fs.c:878-910` — `f_globpath()` forwards `&path` into `globpath()`.
- `src/nvim/cmdexpand.c:3596-3633` — `globpath()` concatenates raw path items with the file pattern and calls `ExpandFromContext()`.
- `src/nvim/cmdexpand.c:2711-2722` — `file_in_path` completion also routes into wildcard expansion with `EW_PATH`.
- `src/nvim/path.c:1172-1202` — `expand_in_path()` reuses `globpath()` for path-based expansion.
- `src/nvim/path.c:1235-1251` — backticks are recognized as special wildcard characters.
- `src/nvim/path.c:1297-1305` — special-wildchar patterns are delegated to `os_expand_wildcards()`.
- `src/nvim/os/shell.c:295-327` — backtick regions are left active in the constructed shell command.
- `src/nvim/os/shell.c:353-355` — `call_shell()` executes the command.
- `src/nvim/tag.c:3078-3085` — safer sibling: tag filename expansion explicitly rejects backticks.

## Impact / severity

This is a **moderate-severity command-execution primitive from project content**:

- **Project content arms the payload by default:** modelines are enabled under `--clean` for non-root users.
- **No plugin or custom config is required to seed the dangerous state:** opening the file is enough to store the attacker-controlled buffer-local `'path'` value.
- **The eventual sink is a core shell-backed wildcard path, not an explicit shell API:** callers that treat `&path` as directory data can unknowingly execute backticks.
- **Built-in expansion plumbing is affected:** explicit `globpath(&path, ...)` calls are exploitable, and the same raw option value also reaches built-in `file_in_path` completion plumbing (`getcompletion(..., 'file_in_path')` / `-complete=file_in_path`).

This is weaker than an “open file and immediate exec” bug, because a later lookup/completion action still has to consult `&path`. But it is materially stronger than a purely self-inflicted footgun, because the attacker-controlled code lives in ordinary file content, rides a default-on trust bridge (modeline), and is later reinterpreted by core filename-expansion code that is supposed to be consuming directory data.
