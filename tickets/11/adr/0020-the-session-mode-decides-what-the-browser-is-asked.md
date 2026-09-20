# 0020. The session's mode decides which tool calls the browser is asked about

Status: accepted, supersedes 0006
Date: 2026-09-20
Ticket: 11
Block: b1

## Context

ADR 0006 put a `PreToolUse` hook in front of every tool call that always
answers `ask`, so the browser sees each one whatever the rules and the mode
say. That was right while the browser had no way to say anything else: a web
page with no authentication in front of it must not be able to start
unapproved work, and a mode the developer could not see or change was a
setting nobody had chosen.

This ticket gives the developer the mode. It can be switched in the chat, the
way it is switched in the terminal, and it is shown under the input at all
times. That changes the premise of 0006: the modes other than `default` are
now the developer saying in advance what need not be asked, made in the same
browser that would otherwise be asked.

The always-ask hook cannot stay as it is once that choice exists. It runs
before the permission rules, so it overrides them: a developer switching to
`acceptEdits` would keep being asked about every edit, and an allow rule they
created through "always allow" would never take effect. The mode would be a
control that visibly does nothing.

The keel workflow's own guarantee does not depend on this hook. What must not
happen unnoticed is a change to a frozen contract, and that is enforced by
the workflow's hooks in the ticket repository, not by the permission gate.

## Options

### Keep 0006 and reimplement the modes on the server

The hook keeps answering `ask`, and keel-web decides for itself what each
mode means: which tools count as an edit, what `dontAsk` denies, what `plan`
blocks. The gate stays the single point every call passes.

Testable without a real agent, and it keeps the property 0006 was written
for. It costs a second implementation of semantics the agent already has, in
a place where drifting from it is silent: the agent can change the mode by
itself, and a developer who reads "acceptEdits" in the terminal and in
keel-web would be reading two different things.

### Let the agent's mode decide, and ask only in `default`

The hook reads the mode it is given and forces the ask only in `default`. In
every other mode it returns no decision, so the agent resolves the call the
way it resolves it in the terminal, and `canUseTool` is reached exactly when
the agent would have prompted.

One source of truth for what a mode means, and the browser behaves like the
terminal. It gives up the "every call is seen" property, which is the point
of the change rather than a side effect. It also cannot be tested against the
types alone: what each mode auto-allows is the agent's behaviour, not a
declaration, so it needs a run against the real agent.

### Offer no mode at all

Leaves 0006 untouched and drops the mode from the ticket. It keeps the
strongest property and fails the ticket's goal, which is that the web chat
feels like Claude Code.

## Decision

The agent's mode decides. The hook asks only in `default`, and the four modes
keel-web offers are `default`, `acceptEdits`, `plan` and `dontAsk`.

`bypassPermissions` and `auto` are not offered and cannot be named on the
wire. The first would run everything unasked, which is the one thing a page
with no authentication in front of it must not be able to turn on. The second
would let a model approve in the developer's place, which is not a choice
this ticket was asked to give away.

"Always allow" is the agent's own session rule: the suggestions it hands to
`canUseTool` are returned as `updatedPermissions`, with every destination
forced to `session`, so nothing is written into the developer's settings
files. Where the agent says a rule would grant more than the call itself, the
offer is not shown at all.

## Consequences

A tool call can now run without the browser having seen it, and that is
visible in the status line rather than implied. `default` remains the mode a
session starts in, so nothing changes for a developer who never touches the
control.

The permission gate stops being the place where every call passes. Its
contract says so, and a reader who finds a tool result with no matching
permission request in a transcript can tell from the mode why.

keel-web now depends on the agent's mode semantics staying what they are. A
change on that side changes what keel-web does, without keel-web changing.
That is the price of not owning a second copy of them.

The behaviour cannot be fully verified from the installed types. Step 9 has
to check against a real agent that a mode switch actually stops the asking,
because a hook that keeps overriding the mode would leave the control looking
right and doing nothing.
