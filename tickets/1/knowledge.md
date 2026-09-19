# 1: Initialize project

Source: https://github.com/NavilaLabs/keel-web/issues/1

## Goals
- [x] keel-web (web UI for keel) has a chosen, documented tech stack that can carry the v1 goals (browser chat with Claude Code, read-only ticket/PR/LikeC4 views, GitHub/Jira/Bitbucket sources, keel phase progress)  `agreed`
- [x] A developer can clone the repo and work in a reproducible devcontainer (runtime + tools only, local Claude Code instance mounted into the container)  `agreed`
- [x] A runnable project skeleton exists, created from a starter template  `agreed`
- [x] A short English README explains what keel-web is  `agreed`

## Problems
- Tech stack is undecided; no constraints given. Preference: keep it simple, leaning Deno + Fresh, still open  `open`
- UI design is explicitly undecided (out of scope here; stack should not force a look)  `open`
- Ticket has no acceptance criteria  `open`
- v1 features are not split into their own tickets; this ticket is scaffolding only (no CI, no backend skeleton)  `resolved: scope confirmed by developer`

## Open questions
- Which starter template? (depends on stack choice, step 6)  `open`
- How exactly is the local Claude Code instance mounted into the devcontainer (config dir, credentials, binary)?  `open`

## Theme blocks
- **b1** Project scaffolding (stack, devcontainer, starter template, README) - `pending`

## Log
- 2026-09-19 - Ticket body was updated after the step 2 consultation (v1 goals, README step added); jump from 2 to 1 (missing_context), step 1 repeated.
