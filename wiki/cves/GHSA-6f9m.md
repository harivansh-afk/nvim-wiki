# GHSA-6f9m-hj8h-xjgj

> Tier: **A** (with caveat) · CVSS 6.6 (CWE-22) · Affected: Neovim 0.5–0.8.2 · Patched: 0.8.3
> Source: [advisory](https://github.com/neovim/neovim/security/advisories/GHSA-6f9m-hj8h-xjgj)

Treesitter injection language name like `../../exploit` resolves through `package.cpath` and `dlopen`s an attacker-planted `.so`. Trigger: open a markdown file (with nvim-treesitter installed) plus a downloadable `.so`. **Caveat:** requires the canonical third-party plugin — accepted anyway because nvim-treesitter is the de-facto Treesitter integration. This sets the upper-flexibility of the bar: "reasonable plugin + reasonable user behavior", not vanilla `nvim --clean` only.

## See Also

- [severity.md §RCE](../severity.md#examples)
- [reachability.md §self-check](../reachability.md#self-check)
- [issue #23579 — WASM plugin sandboxing](https://github.com/neovim/neovim/issues/23579)
