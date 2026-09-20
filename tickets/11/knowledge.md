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
- Is bypassPermissions offered?  `answered: no. auto is offered, see ADR 0022.`
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
- 2026-09-20 - Step 8 for b1 found that ADR 0020's mechanism for "always allow" is not enough on its own: in the default mode the always-ask hook runs before the permission rules, so the agent's session rule would never be reached and the developer would keep being asked. The registry therefore also remembers the tool names a rule now covers and has the hook step aside for them, leaving the matching to the agent. No contract changed, so this is not a jump.
- 2026-09-20 - Step 9 for b1: 253 tests pass, and all seven frozen contracts still carry their step 7 fingerprint. What cannot be tested without a real agent is still open: whether a hook that returns no decision really lets acceptEdits and dontAsk resolve a call.
- 2026-09-20 - Jump from 9 to 7 for b1: the developer asked for the auto mode, which ADR 0020 had left out on my own reasoning rather than theirs. `SessionMode` gains it, so a frozen contract changed. ADR 0022 records the change and why auto is not the same kind of decision as bypassPermissions: a call the classifier is unsure about still reaches the browser.
- 2026-09-20 - b1 is back through 8 and 9 with the five modes: 254 tests pass, and the six contracts the jump did not touch still carry their original fingerprint.
