# CVE / Advisory Library

> Sources: [Severity](../severity.md); [Reachability](../reachability.md)
> Updated: 2026-04-24

Worked examples. Read the cards relevant to your task before forming a hypothesis. Tier label = upstream advisory or maintainer verdict, not the original reporter's claim.

## Accepted (Tier A)

| Card | One-line |
|---|---|
| [CVE-2019-12735](CVE-2019-12735.md) | Modeline `:source!` → arbitrary OS command on file open. Default config. |
| [GHSA-3r8v](GHSA-3r8v.md) | Symlink race in `:write` backup-copy → arbitrary file overwrite. Found by Devin AI. |
| [GHSA-6f9m](GHSA-6f9m.md) | Treesitter language-name path traversal → load malicious `.so`. Plugin-required, accepted anyway. |

## Rejected (Tier C)

| Card | One-line |
|---|---|
| [CVE-2025-26603](CVE-2025-26603.md) | Vim UAF pattern-matched onto Neovim. Precondition (register aliasing) doesn't exist in Nvim. Reporter retracted. |
| [CVE-2024-41965](CVE-2024-41965.md) | Vim double-free pattern-matched. `fname_expand()` semantics in Nvim guarantee `b_sfname` ≠ `b_ffname`. Reporter retracted. |
| [modelineexpr-callback](modelineexpr-callback.md) | Modeline → user-registered callback. Requires `'modelineexpr'` (opt-in, deprecation-tracked). |

## See Also

- [Severity](../severity.md)
- [Reachability](../reachability.md)
- [Runtime instructions](../runtime.md)
