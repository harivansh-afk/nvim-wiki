# Task brief navigation

> Sources: [Runtime instructions](../runtime.md); [Tar plugin extraction path](../tasks/01-tar-extract.md); [Modeline execution pipeline](../tasks/05-modelines.md); [MessagePack-RPC unpacker](../tasks/10-rpc-unpacker.md); [Terminal TermRequest bridge](../tasks/13-termrequest.md); [Swap recovery parser](../tasks/16-swap-recovery.md); [Statusline mini-language parser](../tasks/19-statusline-parser.md)
> Updated: 2026-04-23

## Overview

The runtime instructions and task briefs define a consistent operating model for agents: read the assigned task first, understand the exact code section it names, then use the brief’s references to move outward into source files, docs, and adjacent subsystems. The fastest path is not "read the whole codebase" but "read the exact brief, then expand only into the linked area guide and raw source files."

## Runtime instructions in practice

- Start with [runtime instructions](../runtime.md).
- Read the assigned file under `wiki/tasks/` before opening source files.
- Treat the task brief as the primary statement of scope, risk, and expected investigation direction.
- Use the brief to decide which files, options, and docs matter before branching into related areas.

## How to read a task brief

- `Overview`: names the trust boundary and explains why it matters.
- `Relevant options and knobs`: lists the runtime-controlled inputs or policy switches that shape the surface.
- `Relevant files`: gives the first code and doc files to open.
- `Big-picture references`: points to the runtime docs that explain semantics or subsystem architecture.
- `Recent fix / history signal`: highlights regression history and the exact bug classes already seen nearby.
- `Audit focus`: gives concrete review questions and test directions.
- `See Also`: shows the next subsystem to read if the data flow crosses boundaries.

## Navigation by codebase area

- [Runtime plugin and shell surfaces](../codebase/runtime-plugin-and-shell-surfaces.md): tasks 01 to 03
- [Tag and command surfaces](../codebase/tag-and-command-surfaces.md): tasks 04, 09, and 20
- [Trust and local config surfaces](../codebase/trust-and-local-config-surfaces.md): tasks 05 to 08
- [RPC and channel surfaces](../codebase/rpc-and-channel-surfaces.md): tasks 10 to 12
- [Terminal and TUI surfaces](../codebase/terminal-and-tui-surfaces.md): tasks 13 to 15
- [Persistence and parser surfaces](../codebase/persistence-and-parser-surfaces.md): tasks 16 to 18
- [Display and formatting surfaces](../codebase/display-and-formatting-surfaces.md): task 19

## Suggested reading order for an assigned task

1. Read the assigned task brief.
2. Open the linked area guide for the broader subsystem.
3. Open the raw source files linked from the brief and the guide.
4. Read the runtime docs named in `Big-picture references` only after you know which execution path you are tracing.
5. Follow `See Also` only when the data flow genuinely crosses into a neighboring subsystem.

## See Also

- [Runtime plugin and shell surfaces](../codebase/runtime-plugin-and-shell-surfaces.md)
- [Tag and command surfaces](../codebase/tag-and-command-surfaces.md)
- [Trust and local config surfaces](../codebase/trust-and-local-config-surfaces.md)
- [RPC and channel surfaces](../codebase/rpc-and-channel-surfaces.md)
- [Terminal and TUI surfaces](../codebase/terminal-and-tui-surfaces.md)
- [Persistence and parser surfaces](../codebase/persistence-and-parser-surfaces.md)
- [Display and formatting surfaces](../codebase/display-and-formatting-surfaces.md)
