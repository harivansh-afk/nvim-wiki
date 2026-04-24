# Overlong `'helpfile'` truncation retargets help lookup into a truncated ancestor `tags` file

## Broken invariant

The help fallback in `get_tagfname()` is only supposed to derive one extra help tag file from the configured help root:

> if help lookup falls back to `'helpfile'`, the derived tags path must stay bound to the real `dirname(helpfile)` (or fail closed), not to a shorter alias created by fixed-size truncation.

Current Neovim violates that invariant. The recent fixes for `helpfile` overflows reserve room for `"tags"`, but they still truncate `p_hf` into a fixed `MAXPATHL` buffer before calling `path_tail()` and rewriting the tail to `"tags"`. When the configured helpfile path is longer than 4091 bytes, the rewrite uses the **last slash that survived truncation**, not the real last slash from `helpfile`. The fallback therefore becomes `dirname(truncate(helpfile))/tags`, i.e. a shorter ancestor path chosen by buffer length rather than by policy.

In the practical Linux repro below, `:help pwntag` opens a valid 4093-byte helpfile path, but the later tag discovery fallback searches a shorter ancestor `.../tags` file and reaches the default `term://` `BufReadCmd` sink, resulting in command execution.

## Input -> normalization -> policy gate -> sink

1. **Input:** attacker-controlled `'helpfile'` plus attacker-controlled filesystem contents under the truncated ancestor directory.
   - `src/nvim/help.c:171-200` verifies the raw `p_hf` path exists and opens the help window.
   - `src/nvim/help.c:215` then calls `do_tag(..., DT_HELP, ...)` for the requested help subject.
2. **Normalization drift:** `get_tagfname()` derives the fallback tagfile from a truncated copy of `p_hf`.
   - `src/nvim/tag.c:2496-2508` copies `p_hf` with `xstrlcpy(buf, p_hf, MAXPATHL - STRLEN_LITERAL("tags"))`, then rewrites `path_tail(buf)` to `"tags"` and runs `simplify_filename(buf)`.
   - `src/nvim/path.c:102-118` shows `path_tail()` uses the last separator still present in the already-truncated buffer.
   - So an overlong `helpfile` does **not** become `dirname(helpfile)/tags`; it becomes `dirname(truncated-prefix)/tags`.
3. **Policy gate:** the fallback path is only checked for lexical duplication against `runtimepath` results.
   - `src/nvim/tag.c:2510-2513` compares with `strcmp()` against discovered runtimepath tagfiles.
   - There is no check that the rewritten path still names the real helpfile directory, and no fail-closed behavior when truncation changed the directory.
   - `src/nvim/tag.c:2172-2174` then opens the derived path with `os_fopen(st->tag_fname, "r")`.
4. **Execution sink:** once the wrong tag file is accepted, help-tag filenames reach the same active buffer-open path as ordinary tags.
   - `src/nvim/tag.c:2817-2827` expands the tag filename and accepts any non-file name that matches `BufReadCmd`.
   - `src/nvim/tag.c:2897-2900` passes the accepted filename to `getfile()`.
   - `runtime/lua/vim/_core/defaults.lua:575-581` registers the default `term://*` `BufReadCmd` handler, which runs `jobstart()`.

## Why this breaks the documented contract

- `runtime/doc/helphelp.txt:307-309` documents a strict split:
  - no-argument `:help` opens `'helpfile'`;
  - `:help {subject}` searches `"doc/tags"` files from `'runtimepath'`.
- `runtime/doc/options.txt:3454-3456` further says `'helpfile'` names the main help file and that help files should live together in one directory.
- Nothing in the public contract says that an overlong `'helpfile'` may silently widen `:help {subject}` into an ancestor directory selected by `MAXPATHL` truncation.

So the implementation promise is not merely “avoid overflowing `buf`”; it is “preserve the configured help-root boundary while deriving the fallback tags path.” The current code preserves memory safety but not that boundary.

## Minimal reproducer

This repro was verified on the box with `nvim --clean --headless` (NVIM v0.12.1).

### Commands

```bash
rm -rf /tmp/tfdexec /tmp/tagfile-discovery-proof

python3 - <<'PY'
from pathlib import Path
import shutil, textwrap

root = Path('/tmp/tfdexec')
shutil.rmtree(root, ignore_errors=True)
root.mkdir(parents=True)

# Build a real helpfile path that is valid on Linux but longer than the
# 4091-byte copy limit used by get_tagfname().
prefix = 'tmp/tfdexec/'
segments = ['a' * 255] * 15 + ['b' * 239]
helpfile = prefix + '/'.join(segments) + '/h'
assert len(helpfile) == 4093

# Emulate the vulnerable fallback:
#   xstrlcpy(buf, p_hf, MAXPATHL - strlen("tags"))
#   STRCPY(path_tail(buf), "tags")
copied = helpfile[:4091]
fallback = copied[:copied.rfind('/') + 1] + 'tags'

(Path('/') / helpfile).parent.mkdir(parents=True)
(Path('/') / helpfile).write_text('real help file\n')

(Path('/') / fallback).parent.mkdir(parents=True, exist_ok=True)
(Path('/') / fallback).write_text(
    "pwntag\tterm://sh -c 'printf exploited-help-chain >/tmp/tagfile-discovery-proof'\t1\n"
)

root.joinpath('repro.vim').write_text(textwrap.dedent(f'''
    let &helpfile = '{helpfile}'
    silent! help pwntag
    sleep 600m
    qa!
''').lstrip())
PY

cd /
nvim --clean --headless -S /tmp/tfdexec/repro.vim
cat /tmp/tagfile-discovery-proof
```

### Expected vs actual

- **Expected:** once `'helpfile'` is set to `tmp/tfdexec/.../h`, any helpfile-derived fallback should search `tmp/tfdexec/.../tags` in the **same directory as that helpfile**, or fail if the path is too long. It should never consult a shorter ancestor directory just because the internal copy buffer ran out of room.
- **Actual:** `get_tagfname()` truncates the copied helpfile path, rewrites the last surviving separator to `"tags"`, opens the ancestor `.../tags`, and accepts the attacker-controlled `term://...` tag target. The proof file contains:

```text
exploited-help-chain
```

## Affected files

- `src/nvim/help.c:171-200` — raw `'helpfile'` existence check and help-window open.
- `src/nvim/help.c:215` — `:help` enters the help tag pipeline with `do_tag(..., DT_HELP, ...)`.
- `src/nvim/tag.c:2496-2513` — helpfile-derived fallback path construction and weak duplicate check.
- `src/nvim/path.c:102-118` — `path_tail()` chooses the last separator from the already-truncated buffer.
- `src/nvim/tag.c:2172-2174` — derived tagfile path opened with `os_fopen()`.
- `src/nvim/tag.c:2817-2827` — tag filename expansion plus `os_path_exists || has_autocmd(BufReadCmd)` policy gate.
- `src/nvim/tag.c:2897-2900` — accepted filename handed to `getfile()`.
- `runtime/lua/vim/_core/defaults.lua:575-581` — default `term://*` `BufReadCmd` sink via `jobstart()`.
- `runtime/doc/helphelp.txt:307-309` and `runtime/doc/options.txt:3454-3456` — user-visible help discovery contract.

## Impact / severity

This is a **real help-command execution chain**, but it is **not default-reachable** in the same way as the stock `./tags` bugs:

- **Precondition:** an attacker must be able to influence `'helpfile'` to a path just under `PATH_MAX` and control files in the truncated ancestor directory.
- **Impact once reached:** `:help {subject}` stops respecting the configured help-root boundary and can be steered into the same active `BufReadCmd` sinks as ordinary tag jumps, including the default `term://*` handler under `--clean`.
- **Why it still matters:** this is exactly the same `helpfile` seam that already needed two recent security fixes. The memory-safety invariant was restored, but the policy invariant was not: truncation still changes *which directory* gets trusted.

So the right severity argument is **medium** rather than high: the reachability is narrower than project-local `./tags`, but the resulting primitive is still command execution in core help handling once an attacker controls the overlong `helpfile` path.
