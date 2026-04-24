## runtime instructions

- you must check `/tasks/{task}`
- your job is to find a verifiable and reproducible exploit in the section of code that you have been assigned
- you can use the `tmux subagents` skill to use subagents; you should leverage them to parallelize research and use context carefully

## neovim bug-finding philosophy

1. Start from `wiki/tasks`, not the repo tree: the right unit is a boundary, not a directory.
2. Hunt the seam where attacker-controlled bytes become a more privileged thing: command, path, option, callback, file write, RPC event, parser artifact, or editor state.
3. The best Neovim bugs usually are not deep algorithm bugs; they are boundary bugs, policy bugs, or lifetime bugs.
4. Rank surfaces by four signals: attacker control, default reachability, hand-rolled normalization or parsing, and a dangerous sink.
5. The highest-yield sinks are shell execution, Ex or `:source` execution, filesystem writes, network exposure, callback registration, parser loading, and state deserialization.
6. The codebase parts that matter most are boundary-heavy C modules in `src/nvim/`, Nvim-owned Lua policy in `runtime/lua/vim/`, generated metadata in `options.lua` / `ex_cmds.lua` / `auevents.lua` / `eval.lua`, runtime plugins in `runtime/`, and contract docs in `runtime/doc/`.
7. Read docs first when auditing a surface: `options.txt`, `editing.txt`, `api.txt`, `terminal.txt`, `tagsrch.txt`, `vim_diff.txt`, and `dev_arch.txt` tell you what promise the implementation is supposed to keep.
8. Then read metadata before code: many Neovim invariants are declared in tables and later assumed by handwritten C or Lua.
9. Then trace the handoff chain: input -> normalization -> policy gate -> execution or mutation.
10. Historical exploits were usually chains, not isolated parser accidents.
11. Modeline bugs chained file text -> option metadata (`P_MLE` / `modelineexpr`) -> missing secure check -> callable API -> execution.
12. Archive bugs chained archive member name -> partial path validation -> normalization gap -> shell or extract helper -> arbitrary write or traversal.
13. Tag bugs chained tag file text -> filename expansion or backticks -> shell-adjacent behavior or Ex execution.
14. Treesitter bugs chained injected language text -> parser lookup -> shared-object loading.
15. `exrc` and trust bugs chain local discovery -> path or hash trust decision -> actual executed file.
16. RPC and terminal bugs chain raw bytes -> parser -> queued work -> callback, reentrancy, or lifetime problems.
17. Zero-days in codebases like this are usually found by variant analysis, not magic.
18. Read a real fix, name the invariant it restores, then look for siblings where the same invariant is still missing.
19. After a `zip.vim` traversal fix, audit `tar.vim`, Windows branches, archive siblings, and every helper that still validates pre-normalized paths.
20. After a modeline or secure-mode fix, audit every option or API with similar callback or execution semantics.
21. The most productive general bug classes here are metadata drift, validator or executor drift, normalization or use drift, reentrancy, and teardown.
22. Metadata drift means the table says “restricted” but some real codepath forgets the matching gate.
23. Validator or executor drift means the checker approves one representation and the sink later acts on a different one.
24. Normalization or use drift means the code validates a raw path, name, or handle, then later uses a normalized or aliased object.
25. Reentrancy bugs matter more in Neovim than in many C projects because autocommands, Lua, and RPC can run user code almost anywhere.
26. Any function that fires callbacks before pinning object lifetimes is a prime UAF candidate.
27. Any feature that looks passive but auto-loads, auto-sources, auto-extracts, or auto-parses is a high-priority audit target.
28. Treat “open a file” as the sacred boundary: if merely reading untrusted content can execute code, load a parser, or write to disk, that is the strongest bug class.
29. For Neovim, the most important code-reading heuristic is to prefer representation changes over call depth.
30. Ask where bytes stop being bytes, where paths stop being strings, where options stop being data and become code, and where queued work stops being data and starts mutating state.
31. Distinguish Vim-derived code from Nvim-owned code because they fail differently.
32. Vim-derived code more often hides fixed buffers, pointer arithmetic, shell quoting mistakes, and old “secure mode” edge cases.
33. Nvim-owned code more often hides async queue ordering, ownership transfer, API contract drift, and callback-lifetime bugs.
34. The most dangerous places are mixed seams where Vim-style synchronous parsers feed Nvim-style Lua policy, async dispatch, or callbacks.
35. Use sanitizers for memory safety, but use malicious harnesses for logic bugs: hostile autocmds, weird paths, malformed state files, giant terminal sequences, and parser artifacts.
36. When a bug looks “just weird,” force yourself to state the broken promise: what input crossed what trust boundary into what impact without what consent.
37. That is how you turn a crash or odd behavior into a real vulnerability argument.
38. For bounty-quality research, defaults and scope matter more than edge-case cleverness.
39. Prefer default-on surfaces, core behavior, and exploits that do not require prior trust or niche plugin setups.
40. Do not assume a Vim advisory automatically applies to Neovim; prove reachability in the fork you are auditing.
41. The current `wiki/tasks` breakdown is the right map because it already organizes the codebase by exploit shape.
42. Use it as a launchpad: boundary page -> docs contract -> metadata table -> entrypoint -> handoff -> sink -> sibling variants.
43. If you keep that workflow, every section of the codebase becomes approachable.
44. Highest-value missing surfaces to add next are runtimepath or ftplugin loading, undo-file parsing, quickfix or `errorformat`, autocmd deferral, outgoing UI events, and extmark or decoration lifetime.
45. Those are not special cases; they are more examples of the same seam-first method.
46. The master question is always: what untrusted input became what more dangerous thing, through which normalizer or policy seam, and where else does the same seam exist?

