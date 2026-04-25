# GHSA-3r8v-crjq-wphr

> Tier: **A** · CVSS 6.1 (CWE-59 + CWE-367) · Affected: Neovim ≥ 0.7.0
> Source: [advisory](https://github.com/neovim/neovim/security/advisories/GHSA-3r8v-crjq-wphr). **Found by Devin AI.**

TOCTOU between `os_remove(*backupp)` and `os_copy(...)` in `buf_write_make_backup()`: attacker with write access to the backup dir replaces the path with a symlink between the two calls and Nvim overwrites the symlink target. Defaults trigger this on any symlinked/hardlinked file.

## See Also

- [severity.md §Privesc](../severity.md#examples)
- [reachability.md §self-check](../reachability.md#self-check)
