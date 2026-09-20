# 11: The Claude Code experience: autocomplete and model selection

## Goals
- [x] Typing `/` in the chat offers slash commands with description and argument hint, and the list stays current when the SDK reports a change.  `agreed`
- [x] Typing `@` offers files and directories of the workspace: tracked files plus untracked files that are not ignored, cached per workspace and filtered on the server.  `agreed`
- [x] The model can be chosen and changed mid-session, together with the effort level.  `agreed`
- [x] The permission mode can be switched in the chat: default, acceptEdits, plan, dontAsk. bypassPermissions is not offered.  `agreed`
- [x] A status line under the input shows model, mode and effort and lets the developer change them.  `agreed`
- [x] The developer can allow a tool for the rest of the session ("always allow").  `agreed`
- [x] The input keeps a history (arrow up) and a draft per ticket.  `agreed`
- [x] The questions tool is answered with the mouse: larger click areas, a free text "Other", direct send on a single choice, and a way to decline.  `agreed`
- Out of scope: plan approval, todo list, context and cost display, image and file paste, rewind.

## Problems
- The chat input is a plain textarea without popup, completion, model or mode control.  `open`
- `RunQuery` is deliberately narrower than the SDK `query`, so `supportedCommands`, `supportedModels`, `setModel` and `setPermissionMode` are not reachable.  `open`
- The wire contract has no types for command lists, model, mode or file references.  `open`
- The server has no file listing. `directories` lists subdirectories only and skips dotfiles.  `open`
- The `PreToolUse` hook always returns `ask`, so a mode switch would have no effect on the gate.  `open`
- Two contracts frozen from documentation in ticket 4 were wrong against the installed types.  `open: read the .d.ts before freezing anything in this ticket`

## Open questions
- Can `supportedCommands()` be called before the first turn, or only after the SDK's `init`?  `open`
- How do mode, "always allow" and the always-ask hook interact in detail?  `open`
- Are the selections stored per session with the last value as the workspace default?  `answered: per session, the last value is the default for new sessions in the workspace`
- Which files does `@` offer?  `answered: tracked plus untracked and not ignored, plus directories`
- Is bypassPermissions offered?  `answered: no`
- What does "questions tool nicely integrated" mean?  `answered: mouse first, Other, direct send on a single choice, decline`

## Theme blocks
- **b1** Session controls: slash command list, model, effort, mode, always allow, status line - `pending`
- **b2** Composer: slash and @ popup, file index, keyboard, input history and draft - `pending`
- **b3** Questions tool: mouse-first answering, Other, decline - `pending`

Risk: b2 consumes the command list from b1 as a wire event, so b1's contract must be settled first. `chatApi` and `protocol` are shared. The file route goes into its own component to avoid overlapping claims.

## Log
- 2026-09-20 - Scope agreed in step 2, blocks cut in step 4 (three blocks).
