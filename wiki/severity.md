# Severity

> Sources: [Reachability](reachability.md); [CVE library](cves/index.md); [Runtime instructions](runtime.md)
> Updated: 2026-04-24
> Scope: codebase-agnostic. Neovim-specific carve-outs live in [reachability.md](reachability.md).

A finding must clear two filters before `findings.md` is committed: this file (is it a real vuln class?) and [reachability](reachability.md) (does it actually fire on default Neovim?). Failing either → write [`negatives.md`](reachability.md#negativesmd) instead.

## Bug class + primitive

A finding needs both. "Memory corruption" without a primitive is a crash; "arbitrary write" without a class is undefined behavior.

| Class | Primitive required |
|---|---|
| RCE / LCE | Concrete chain to attacker-controlled execution: shell exec, `:source`, `dlopen`, callback hijack. PoC must observably run code (`touch /tmp/pwn` etc). |
| Memory corruption | Exploitability argument, not a crash. Controlled offset *and* value for OOB; reachable reuse for UAF; triggerable cast for type confusion. |
| Path traversal / arbitrary write | Demonstrate writing outside the intended scope with attacker-influenced contents or path. |
| Info disclosure | Read of bytes the attacker should not see (creds, other-tenant data, ASLR leaks used to bypass). Reading your own buffer doesn't count. |
| Privesc | Action at higher trust than the attacker had — e.g. another user's Nvim writes that user's files via attacker-planted bytes. |
| DoS | Tier-C by default. Only a finding if it bricks state (`:trust` db, ShaDa, swap) or is unrecoverable without filesystem intervention. |

## Tier rubric

Tier is determined by the [3-question reachability test](reachability.md#self-check) applied to the bug class:

| Tier | Criteria | Action |
|---|---|---|
| **A** | Real class **and** Q1=default-on **and** Q2=minimal-action **and** Q3=no-prior-capability. | Commit `findings.md`. |
| **B** | Real class but one of Q1/Q2/Q3 fails *and* the primitive is **novel**. | Keep searching. Only commit if the primitive is genuinely new. |
| **C** | Real class but Q1/Q2/Q3 fails and the primitive is well-known. Or working-as-documented behavior. | Reject. Write `negatives.md`. |
| unclear | Cannot answer Q1/Q2/Q3 from the source. | Treat as B until proven A. Don't guess. |

## Stop rule

Tier B/C is **not partial credit** — it's noise the next agent has to re-disprove. If your best candidate is not Tier A, write [`negatives.md`](reachability.md#negativesmd) and exit.

## Examples

One per class. Generic shapes; Neovim-specific cases live in [`cves/`](cves/index.md).

- **RCE** — ✅ open file → shell exec ([CVE-2019-12735](cves/CVE-2019-12735.md)). ❌ user runs `:source attacker.vim` (feature).
- **Memory corruption** — ✅ untrusted file's parser metadata controls bytes that reach a fixed-size sink. ❌ crash on malformed input the parser already rejects.
- **Path traversal** — ✅ archive member name with `../` escapes extraction dir. ❌ user runs `:write /etc/passwd` (their command).
- **Info disclosure** — ✅ reading file leaks bytes from another user's heap. ❌ `:%y +` copies buffer to clipboard (feature).
- **Privesc** — ✅ symlink race lets attacker overwrite root-owned files when root edits in attacker-writable dir ([GHSA-3r8v](cves/GHSA-3r8v.md)). ❌ attacker with vimrc write executes code (vimrc *is* code).
- **DoS** — usually ❌. Mere hangs/crashes aren't findings; only Tier-A if it corrupts on-disk trust/swap/ShaDa state.

## See Also

- [Reachability](reachability.md) — Neovim-specific bar
- [CVEs](cves/index.md) — worked examples, accepted and rejected
- [Runtime instructions](runtime.md)
