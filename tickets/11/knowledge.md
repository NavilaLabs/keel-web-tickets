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
- Can `supportedCommands()` be called before the first turn, or only after the SDK's `init`?  `answered: yes, it awaits the initialisation started in the Query constructor. The registry's comment about the agent announcing itself applies to system/init only.`
- How do mode, "always allow" and the always-ask hook interact in detail?  `answered: the hook asks only in default mode, see ADR 0020. Always allow returns the agent's own suggestions as updatedPermissions with every destination forced to session.`
- Does a hook that returns no decision really let acceptEdits and dontAsk resolve a call, and is permission_mode current in the hook right after setPermissionMode?  `open: not knowable from the types, has to be checked against a real agent in step 9`
- Does the SDK expand a plain `@path` in a message the way the CLI's prompt handler does?  `open: to be checked with a real turn in b2`
- Which previewFormat does an AskUserQuestion request actually arrive with? The docs say absent when unset, the installed types say markdown is the default.  `open: to be observed on a real request in b3`
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
- 2026-09-20 - Step 6 for all three blocks. b1: the agent's own mode decides what the browser is asked, which supersedes ADR 0006. b2: git-based file index, custom listbox, no row highlighted by default. b3: hybrid answer flow with no protocol change.
- 2026-09-20 - Step 7 for b1: contracts frozen in seven declaration files, ADRs 0020 and 0021 written. `chat` is the seam between b1 and b2: neither claims it, because its own responsibility does not change, and each adds one edge into it.
