# 0006. Gate every tool call in the browser with a PreToolUse hook

Status: accepted
Date: 2026-09-19
Ticket: 4
Block: b1

## Context

The web UI lets a browser drive a Claude Code session that runs in the
container with the code repository as its working directory and the ticket
repository mounted next to it. That is execution on the developer's machine,
reached from a page with no authentication in front of it. "Nothing runs
unapproved" is therefore a property the design has to produce, not a default
to inherit.

The Agent SDK resolves a tool call through hooks, deny rules, ask rules,
permission mode, allow rules, and only then the `canUseTool` callback. A call
that any earlier step approves never reaches the callback, and project setting
sources are loaded by default.

## Options

### canUseTool alone

The smallest thing that works, and what the comparable projects do. The
callback renders the prompt in the browser and returns the answer.

Its hole is documented: a bare entry in `allowedTools`, `acceptEdits` mode, or
an allow rule in the repository's own `.claude/settings.json` resolves the call
first and the browser never sees it. A rule committed by anyone, for any
reason, silently widens what the UI may run.

### canUseTool with project setting sources switched off

Passing no setting sources closes the hole by removing the files that could
contain a rule. It also removes project skills, the project CLAUDE.md and
project hooks, which is what makes the session a real Claude Code session and
the keel workflow drivable from the browser at all. It buys safety by deleting
the feature.

### A PreToolUse hook that returns ask, with canUseTool rendering it

The hook runs before every tool call regardless of rules and modes. Returning
`ask` routes the call to `canUseTool`, so the browser decides, and the project's
own settings, skills and hooks keep working.

The cost is a second gate with its own precedence, and one more place that has
to be correct for the guarantee to hold.

## Decision

A `PreToolUse` hook returning `ask`, with `canUseTool` as the browser-facing
callback.

The gate holds a call until the browser answers. There is no timeout: silence
leaves the turn paused rather than being recorded as a refusal, and a viewer
that attaches later sees what is waiting and can answer it. On session close
everything still held resolves as a deny, so no turn is left frozen.

`AskUserQuestion` arrives through the same callback, so the wire protocol
carries structured questions and structured answers rather than a boolean. A
permission modelled as allow-or-deny would make every consultation in the keel
workflow unanswerable from the browser.

The "always allow" affordance is left out. It works by writing a rule into
`.claude/settings.local.json`, which would both mutate the code repository from
the UI and create exactly the bypass this decision exists to prevent.

## Consequences

Every tool call is visible in the browser and in the transcript, including the
ones a terminal session would have auto-approved. This is more prompting than a
terminal, deliberately.

A held call keeps its agent subprocess resident, so an abandoned prompt costs
memory until the session is closed. Sessions need an eviction policy; they do
not time out on their own.

A prompt that was pending when the process died is gone on resume, and the
resumed session does not ask again. Reattaching has to reconcile rather than
assume the prompt is still live.

If a later ticket wants unattended runs, it needs a different decision, not a
setting: this one is what makes the localhost-only posture survivable.
