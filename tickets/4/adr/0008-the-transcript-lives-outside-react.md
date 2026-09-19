# 0008. Keep the transcript outside React and fold it with a pure function

Status: accepted
Date: 2026-09-19
Ticket: 4
Block: b2

## Context

The wire contract dictates more of this than any library would. Recorded events
carry a sequence number that starts at 1 and never reorders; rendering the same
sequence twice must produce the same result; token-level deltas carry no
sequence number, are never replayed, and a client that ignores them entirely
still renders a correct transcript.

So the client holds two different things: an append-only list that is the
record, and a short string that is not. They change at very different rates. A
token arrives many times a second; a recorded event does not.

There is a second problem the contract creates. A stream is opened per ticket
and survives reloads, but React 19's StrictMode runs effects as setup, cleanup,
setup. An effect that opens the stream therefore opens two, attaches two
sessions and replays twice in development.

## Options

### A reducer in context

`(state, event) => state` as a pure function. The most directly testable option
in the block: every claim about replay idempotence, sequence gaps and
tool pairing becomes a plain test with no DOM.

But every token dispatches, and every consumer of the context re-renders. With
a long transcript and markdown, that is the thing that will be visibly slow.
React Compiler is not in this project, so nothing hides it.

### An external store read through `useSyncExternalStore`

Separate subscriptions for the transcript and the draft, so a token redraws one
bubble and leaves the list alone. The store owns the connection, which means
mounting and unmounting a component never opens or closes a stream and
StrictMode stops being a question.

It costs about fifty lines of subscribe and notify, and it brings a rule that is
easy to get subtly wrong: the snapshot must return the identical reference when
nothing changed, or React loops forever.

### A store library such as Zustand

Saves those fifty lines and the identity discipline, at the price of a
dependency and a second vocabulary in a repository that has no state library
and a visible preference for explicit hand-written seams.

### Writing deltas straight to the DOM

Fastest possible, and defensible because the contract calls deltas best-effort.
It makes the delta path untestable without a DOM.

## Decision

An external store with `useSyncExternalStore`, two subscriptions, and the
connection owned by the store rather than by an effect.

The fold from events to render items stays a pure, total function in its own
right. It is the highest-value test in the block, because it is what makes the
contract's promise checkable: fold a replayed sequence and a live one, and the
result must be identical.

Deltas are coalesced before notifying, so a hundred tokens a second do not
become a hundred renders. The draft is a separate trailing element and is
dropped when the finished message arrives, which also means the draft can stay
plain text and only the finished message is parsed as markdown.

## Consequences

`useSyncExternalStore` is a hook most React code never touches, so the store
needs its identity rules stated where the next reader will find them.

Because the store owns the connection, opening and closing it is an explicit
call rather than a lifecycle side effect. Nothing about StrictMode has to be
suppressed, worked around or explained.

The fold being pure means the permission prompt cannot hold its own state about
what was answered: a prompt is retired by the arriving `permission.resolved`
event, not by the click. That is what makes a second browser tab correct for
free, and a 409 from the input endpoint an expected outcome rather than an
error.

Swapping to a reducer or to Zustand later is a change behind the store's own
interface, not a change to the components.
