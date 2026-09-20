# 0022. Offer the auto mode too

Status: accepted, changes the mode list of 0020
Date: 2026-09-20
Ticket: 11
Block: b1

## Context

ADR 0020 settled that the session's mode decides which tool calls reach the
browser, and listed the four modes keel-web would offer: `default`,
`acceptEdits`, `plan` and `dontAsk`. It left out both modes the agent has
beyond those. `bypassPermissions` was left out because it runs everything
unasked. `auto` was left out on the grounds that it lets a model approve in
the developer's place, which the ticket had not asked to give away.

That reasoning was mine, not the developer's, and it was wrong on the facts
of what the ticket is for. The goal is that the web chat feels like Claude
Code, and `auto` is one of the modes a developer cycles through in the
terminal. Leaving it out makes the control incomplete in the one place the
ticket set out to make complete.

It is also not the same kind of decision as `bypassPermissions`. In `auto`
the classifier answers what it is sure about and escalates the rest, and an
escalated call arrives at `canUseTool`, which is the browser prompt. The
developer keeps seeing the calls the agent is unsure about. In
`bypassPermissions` nothing is seen at all, and the agent refuses to run it
without `allowDangerouslySkipPermissions`, which keel-web never passes.

The frozen `SessionMode` names the modes, so this changes a contract that was
frozen at step 7, reached from step 9.

## Options

### Leave 0020's list as it is

Keeps the frozen contract and the ADR intact, and leaves a mode the terminal
has and keel-web does not. The developer asked for it, so this is only an
option in the sense that doing nothing always is.

### Offer `auto` alongside the other four

`SessionMode` gains `auto`, the server accepts it, and the status line lists
it between `default` and `acceptEdits`, which is where it sits on the scale
of how much still reaches the developer.

Nothing else in 0020 changes. The hook already steps aside in every mode but
`default`, so `auto` is resolved by the agent the way the terminal resolves
it, and the calls the classifier escalates still arrive at the browser.

### Offer `auto` and warn about it in the interface

The same, with a line of prose in the status line saying that a model is
answering. It states what the mode already says and costs room in a line
that has to stay short, so the mode's own colour carries it instead.

## Decision

`auto` is offered, as the second entry in the list and in the shift-and-tab
ring. It is coloured like the other modes in which work happens without being
asked about, so the status line shows at a glance that the session is not in
its default state.

`bypassPermissions` stays out, for the reason 0020 gave.

## Consequences

Five modes rather than four. `SessionMode`, the server's two validation lists
and the status line's list all name the same five, and the ordering is
deliberate: it runs from everything reaching the developer to nothing
running at all.

A classifier can now approve a tool call in this session. What it approves is
still recorded as a tool call in the transcript, so the work stays visible,
but the permission step for it is not: there is no `permission.requested` for
a call that was never asked about. The same was already true of `dontAsk`.

A call the classifier denies outright is not recorded at all. The agent
announces those separately, and keel-web does not listen for that message.
That gap is not new with this ADR, but `auto` makes it easier to reach, and
it is worth a later ticket rather than a change here.
