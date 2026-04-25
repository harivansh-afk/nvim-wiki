# Reachability (Neovim)

> Sources: [Severity](severity.md); [CVEs](cves/index.md); [issue #38985 — security model](https://github.com/neovim/neovim/issues/38985); [issue #25811 — remove sandbox/secure/modelineexpr](https://github.com/neovim/neovim/issues/25811); [`runtime/doc/options.txt` (modeline)](../raw/neovim/runtime/doc/options.txt); [`runtime/doc/vim_diff.txt`](../raw/neovim/runtime/doc/vim_diff.txt) (Neovim removed `'secure'`)
> Updated: 2026-04-24

The project-shaped half of the bar. A finding has cleared [severity](severity.md) — now: does it actually fire under default Neovim from a realistic action?

## Threat model (justinmk, [#38985](https://github.com/neovim/neovim/issues/38985))

> "vim/nvim have massive surface area, and almost any action can execute third-party plugin code, and shell commands. There is no meaningful way to make that 'secure'. However, there are some actions that should be safe by default. For example, **we 100% want to avoid security risks from merely *reading* a file or directory**."

Maintainer-tracked surfaces:

| Surface | Status | Implication |
|---|---|---|
| `'exrc'` (`.nvim.lua`/`.nvimrc`/`.exrc`) | ✅ gated by `:trust` | Bypass the consent gate or it's not a vuln. |
| `'modeline'` / `'modelineexpr'` | ⚠️ TODO ([#39044](https://github.com/neovim/neovim/issues/39044), [#25811](https://github.com/neovim/neovim/issues/25811)) | `'modelineexpr'` is opt-in and slated for removal. Findings assuming it on are Tier-C. |
| `require('foo')` / `package.path='./?.lua'` | ⚠️ TODO ([#38966](https://github.com/neovim/neovim/issues/38966)) | Known-broken; variants without new primitive are Tier-C. |
| ftplugins via `'filetype'` | ❓ open | justinmk: "if a ftplugin invokes `system()`, a malicious filename… can invoke shell code. **And this has happened more than once.**" Tier-A territory. |
| `nvim://` URI | tracked ([#38006](https://github.com/neovim/neovim/pull/38006)) | New surface, in scope. |
| Treesitter parser loading | partial fix ([GHSA-6f9m](cves/GHSA-6f9m.md)); [WASM (#23579)](https://github.com/neovim/neovim/issues/23579) is the long plan | Path-traversal variants Tier-B; new attacker-`.so`-loading paths Tier-A. |
| Untrusted plugins | out of scope | "User installed a plugin that does X" is the user's problem. |
| Supply-chain (CI, deps) | out of scope | Don't audit. |

## Self-check

Answer in order. **Stop at the first NO.**

1. **Severity**: real bug class with a real primitive per [severity.md](severity.md)? *NO → not a finding.*
2. **Q1 default-on**: fires on a `nvim --clean` build, no plugins, no manager? *NO → either prove the option is on by default ([options.lua](https://github.com/neovim/neovim/blob/master/src/nvim/options.lua)) or downgrade.*
3. **Q2 minimal action**: trigger is "open / view / clone / `:e` an untrusted file"? *NO →*
   - "user runs `:source`/`:!`/`:set X`": **C** (their command)
   - "user accepts `:trust` prompt": **C** (consented)
   - "user enables `'modelineexpr'`": **C** (opt-in, deprecation-tracked)
   - "user clicks `nvim://` link": **A** (in-scope per #38006)
4. **Q3 no prior capability**: works without pre-existing artifacts on the victim machine? *NO →*
   - "requires user-registered autocmd/callback": **C** (the autocmd is the bug)
   - "requires installed third-party plugin": usually **C**; exception for canonical ones (treesitter, LSP) → **B** at most
   - "requires malicious `--remote-ui` peer the user trusted": **C** (trust relationship)
   - "requires victim on mainstream OS / terminal": **A**
5. **Carve-outs**: does this rely on documented-by-design or out-of-scope behavior?
   - `'secure'` / `:sandbox` / `'modelineexpr'`: slated for removal ([#25811](https://github.com/neovim/neovim/issues/25811)) → **C**
   - `:!`, `system()`, `jobstart()`: features
   - Vim CVE pattern not reachable in Neovim's reimpl: **C** ([rejected examples](cves/index.md#rejected))
   - **A crash is not a CVE** (zeertzjq, [#34044](https://github.com/neovim/neovim/issues/34044#issuecomment-2884975533)): need a chain to a sink

A/A/A and no carve-out → write `findings.md`. Anything else → `negatives.md`.

## negatives.md

```markdown
# <slug> — ruled-out candidates

## Candidate A: <one-line>
- Bug class: <RCE / mem-corruption / …>
- Failed at: Q<1|2|3|carve-out>
- Reason: <one paragraph, cite line numbers>
- What would make it Tier-A: <if escalation is plausible, name it>
```

A `negatives.md` with one honest "I looked, found nothing" entry is a valid agent output.

## See Also

- [Severity](severity.md)
- [CVEs accepted](cves/index.md)
- [CVEs rejected](cves/index.md#rejected)
- [Runtime instructions](runtime.md)
