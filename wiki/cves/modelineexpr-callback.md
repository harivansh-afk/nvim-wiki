# `modelineexpr`-gated callback abuse (rejected class)

> Tier: **C** · default-off feature, slated for removal
> Source: [`runtime/doc/options.txt` *no-modeline-option*](../../raw/neovim/runtime/doc/options.txt) · [issue #25811 — remove sandbox/secure/modelineexpr](https://github.com/neovim/neovim/issues/25811) · [issue #39044 — modeline `:trust`](https://github.com/neovim/neovim/issues/39044)

Findings of the form *"a modeline can set `complete`/`foldexpr`/`tagfunc`/etc to call a user-registered callback"* fail three independent gates: `'modelineexpr'` is opt-in default-off (Q1), it's [slated for removal](https://github.com/neovim/neovim/issues/25811) (carve-out), and the "vulnerable callback" is the victim's own code (Q3). Tier-A modeline findings live in the `filetype` → ftplugin → `system()` chain instead — per justinmk ([#39044](https://github.com/neovim/neovim/issues/39044)): *"if a ftplugin invokes `system()`, a malicious filename… can invoke shell code. And this has happened more than once."*

## See Also

- [tasks/05-modelines.md](../tasks/05-modelines.md)
- [reachability.md §carve-outs](../reachability.md#self-check)
- [CVE-2019-12735](CVE-2019-12735.md) (the canonical Tier-A modeline RCE)
