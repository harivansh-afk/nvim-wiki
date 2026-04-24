# Knowledge Base Index

## agents

Operational guidance for agents using the task briefs and working from the runtime instructions.

| Article | Summary | Updated |
|---------|---------|---------|
| [Runtime instructions](runtime.md) | User-authored runtime rules: read the assigned task brief, work the assigned code section, and use agents thoughtfully. | 2026-04-23 |
| [Task brief navigation](agents/task-brief-navigation.md) | Explains how to read the task briefs, which section to use first, and where to go next by codebase area. | 2026-04-23 |

## runtime-plugins-and-shell

Archive extraction and shell-facing execution surfaces in runtime plugins and core path expansion code.

| Article | Summary | Updated |
|---------|---------|---------|
| [Runtime plugin and shell surfaces](codebase/runtime-plugin-and-shell-surfaces.md) | Maps tar.vim, zip.vim, and shell/path execution into a shared guide for command-construction risks. | 2026-04-23 |
| [Tar plugin extraction path](tasks/01-tar-extract.md) | Tar.vim extraction surface where archive member names reach shell execution and filesystem writes. | 2026-04-23 |
| [Zip plugin extraction path](tasks/02-zip-extract.md) | Zip.vim extraction surface with normalization checks and separate GNU or PowerShell branches. | 2026-04-23 |
| [Shell wildcard and backtick expansion](tasks/03-shell-glob-backtick.md) | Core shell and backtick execution paths in `shell.c` and `path.c`. | 2026-04-23 |

## tag-and-command-surfaces

Tag discovery, tag jump handling, and Ex range parsing around command-facing navigation metadata.

| Article | Summary | Updated |
|---------|---------|---------|
| [Tag and command surfaces](codebase/tag-and-command-surfaces.md) | Connects tag discovery, tag jump expansion, and Ex address parsing into one navigation map. | 2026-04-23 |
| [Tag jump and tag filename expansion](tasks/04-tag-jump-expand.md) | Tag navigation and filename expansion from untrusted tag metadata. | 2026-04-23 |
| [Tag and helpfile discovery](tasks/09-tagfile-discovery.md) | Tag and help path discovery logic in `tag.c`, especially around `helpfile` and `runtimepath`. | 2026-04-23 |
| [Ex address parser](tasks/20-ex-address-parser.md) | Hand-rolled Ex range parsing in `ex_docmd.c` before broader command execution. | 2026-04-23 |

## trust-and-local-config

Modelines, secure-mode gates, project-local config loading, and the trust database that decide when local code runs.

| Article | Summary | Updated |
|---------|---------|---------|
| [Trust and local config surfaces](codebase/trust-and-local-config-surfaces.md) | Maps modelines, secure-mode checks, exrc loading, and trust persistence into a single policy guide. | 2026-04-23 |
| [Modeline execution pipeline](tasks/05-modelines.md) | File-content driven option setting and modeline policy enforcement. | 2026-04-23 |
| [mapset secure-mode gate](tasks/06-mapset-secure.md) | Function-level secure-mode gate around `mapset()` and related restricted contexts. | 2026-04-23 |
| [Project-local exrc loader](tasks/07-exrc-loader.md) | Upward project config discovery and execution for `.nvim.lua`, `.nvimrc`, and `.exrc`. | 2026-04-23 |
| [Trust database and :trust bridge](tasks/08-trust-db.md) | Trust store management and the C/Lua bridge that decides whether local files are allowed to run. | 2026-04-23 |

## rpc-and-channels

Remote parsing, dispatch, and transport setup for MessagePack-RPC, sockets, jobs, and embed mode.

| Article | Summary | Updated |
|---------|---------|---------|
| [RPC and channel surfaces](codebase/rpc-and-channel-surfaces.md) | Tracks the flow from incoming bytes to parsed messages, queued requests, and transport lifecycle. | 2026-04-23 |
| [MessagePack-RPC unpacker](tasks/10-rpc-unpacker.md) | First structured parse of RPC bytes, including redraw-specific fast paths. | 2026-04-23 |
| [MessagePack-RPC dispatch](tasks/11-rpc-dispatch.md) | Request routing, response matching, and fast or deferred execution decisions. | 2026-04-23 |
| [Channel transport creation](tasks/12-channel-transports.md) | Shared setup and teardown for jobs, sockets, PTYs, stdio embed mode, and raw streams. | 2026-04-23 |

## terminal-and-tui

Terminal child output, OSC and TermRequest handling, and host terminal input parsing in the TUI stack.

| Article | Summary | Updated |
|---------|---------|---------|
| [Terminal and TUI surfaces](codebase/terminal-and-tui-surfaces.md) | Splits the terminal stack into child-output control flows and host-input parsing paths. | 2026-04-23 |
| [Terminal TermRequest bridge](tasks/13-termrequest.md) | Terminal control sequences promoted into `TermRequest` events and related side effects. | 2026-04-23 |
| [OSC 52 selection parser](tasks/14-osc52-selection.md) | Clipboard and selection parsing inside the embedded terminal stack. | 2026-04-23 |
| [TUI CSI and raw input parsing](tasks/15-tui-csi-input.md) | Raw terminal input and CSI parsing before bytes become keys, reports, or paste events. | 2026-04-23 |

## persistence-and-parsers

Hostile-file parsers and persisted-state readers for swap recovery, ShaDa, and spell resources.

| Article | Summary | Updated |
|---------|---------|---------|
| [Persistence and parser surfaces](codebase/persistence-and-parser-surfaces.md) | Groups the file-driven parsers that read swap metadata, ShaDa state, and spell resources. | 2026-04-23 |
| [Swap recovery parser](tasks/16-swap-recovery.md) | Recovery-time parsing of untrusted swap metadata in `memline.c`. | 2026-04-23 |
| [ShaDa item parser](tasks/17-shada-parser.md) | MessagePack-based persisted-state parsing and compatibility handling in `shada.c`. | 2026-04-23 |
| [Spell affix parser](tasks/18-spell-affix-parser.md) | `.aff` and `.dic` parsing plus spell generation logic in `spellfile.c`. | 2026-04-23 |

## display-and-formatting

Mini-language and option parsing for statusline-like rendering surfaces.

| Article | Summary | Updated |
|---------|---------|---------|
| [Display and formatting surfaces](codebase/display-and-formatting-surfaces.md) | Shows where statusline-like format strings are validated, rendered, and extended by option metadata. | 2026-04-23 |
| [Statusline mini-language parser](tasks/19-statusline-parser.md) | Statusline, statuscolumn, and rulerformat parsing in `statusline.c` and `optionstr.c`. | 2026-04-23 |

## neovim

Compiled notes from the upstream Neovim repository docs about architecture, build workflows, and project maintenance.

| Article | Summary | Updated |
|---------|---------|---------|
| [Neovim project goals and layout](neovim/project-goals-and-layout.md) | Refactoring goals, source tree layout, and the main extension boundaries between C, Lua, runtime, and UI layers. | 2026-04-23 |
| [Neovim build, install, and test workflow](neovim/build-install-and-test-workflow.md) | How Neovim is installed, built from source, run locally, and verified through its test harness. | 2026-04-23 |
| [Neovim contribution and maintenance workflow](neovim/contribution-and-maintenance-workflow.md) | Contributor expectations, PR and commit conventions, release policy, and long-term maintenance practices. | 2026-04-23 |
