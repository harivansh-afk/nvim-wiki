# Zip plugin extraction path

## Overview

This is another mostly Vim-derived runtime-plugin boundary, but one with a long run of recent hardening. `zip#Extract()` in [`runtime/autoload/zip.vim`](../../../neovim/runtime/autoload/zip.vim) takes the selected archive member name, rewrites it for GNU or PowerShell execution, and extracts it into the working directory. It ranks extremely high because the code repeatedly had to block traversal, absolute-path writes, and shell-adjacent edge cases on different platforms.

## Relevant options and knobs

- `g:zip_extractcmd`: chooses the extraction command path.
- `g:zip_unzipcmd`: affects how archive contents are listed and read.
- `&shell`: matters because the plugin has GNU-shell and PowerShell execution branches.
- `&shellslash`: explicitly manipulated by the plugin during extraction setup.

## Relevant files

- [`runtime/autoload/zip.vim`](../../../neovim/runtime/autoload/zip.vim)
- [`runtime/doc/pi_zip.txt`](../../../neovim/runtime/doc/pi_zip.txt)

## File tree

```text
runtime/
  autoload/
    zip.vim
  doc/
    pi_zip.txt
```

## Big-picture references

- [`runtime/doc/pi_zip.txt`](../../../neovim/runtime/doc/pi_zip.txt): plugin-specific documentation, including a warning to be careful with untrusted input.
- [`runtime/doc/dev_arch.txt`](../../../neovim/runtime/doc/dev_arch.txt): runtime plugins are part of the supported user-facing surface, not just examples.
- [`runtime/doc/vim_diff.txt`](../../../neovim/runtime/doc/vim_diff.txt): useful when deciding whether a bug belongs to inherited runtime logic or newer Nvim wrappers.

## Recent fix / history signal

- `5cfdd4d8b9`: security fix for zip.vim path traversal (#34951).
- `0851ac2706`: security fix for another zip.vim path traversal issue (#38693).
- `c0f33c9a86`: Windows-specific path traversal hardening (#39051).
- `56ed27d718` and `bf084967d7`: fixes around absolute-path writes and extraction target validation (#39094, #38810).

## Audit focus

- Review the exact member-name normalization rules, especially the checks for leading slash, `..`, glob characters, and Windows path oddities.
- Compare GNU and PowerShell code paths; historically the platform split is where mismatched escaping assumptions show up.
- Check whether target-file existence checks and post-extraction validation can still be bypassed by archive naming tricks.
- Treat the extraction target as attacker-controlled even when it comes from a listing buffer rather than a direct function argument.

## See Also

- [Tar plugin extraction path](01-tar-extract.md)
- [Shell wildcard and backtick expansion](03-shell-glob-backtick.md)
