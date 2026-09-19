# 0005. Stream over SSE, with an application-owned event log as the record

Status: accepted
Date: 2026-09-19
Ticket: 4
Block: b1

## Context

The browser must see Claude Code's output as it is produced, and a page reload
must not destroy the run: a returning viewer reattaches to a live session, or
replays the transcript if the run has ended. At the same time a permission
prompt has to travel to the browser mid-turn and its answer has to come back
while the turn is still open.

Two questions had to be answered together, because the answer to one decides
how much work the other is: which transport carries the stream, and what the
replay is built from.

Constraints: Hono on Node with `@hono/node-server` v1, the Vite dev server
proxying `/api`, and a client that is not yet written. No authentication in
front of the UI, so the surface should stay small.

## Options

### SSE plus one input endpoint

`GET` opens an SSE stream, `POST` carries every client-to-server action as a
tagged command. `EventSource` reconnects on its own and sends `Last-Event-ID`,
so resuming after a reload is a slice of a numbered event log rather than a
handshake. Nothing new is installed and the existing proxy configuration keeps
working.

The stream is a viewer, not a channel: it cannot POST or set headers, so every
action needs the second endpoint. Two failure modes need proving rather than
assuming, because both have a history: Hono's `stream.onAbort` on Node, and the
Vite proxy forwarding a client disconnect to the backend.

### WebSocket

One connection carrying both directions, no correlation between two endpoints,
and a disconnect that is unambiguous. `@hono/node-ws` is deprecated, so this
means `@hono/node-server` v2, a major adapter upgrade in the same ticket that
introduces the Agent SDK, plus `ws` and a change to the Vite proxy. Reconnect
and replay are hand-written.

This is what the comparable projects chose. None of them needed replay from a
sequence number, which is the part that is free here and not there.

### Application-owned event log

Normalised events with sequence numbers, appended as they are produced.
Replayed and live events are the same objects, so the client renders one shape.
Permission requests and their outcomes are part of the record. Durability,
rotation and schema evolution are ours.

### The SDK's own session storage

Store only the mapping from ticket to session id and read history back through
the SDK. Durability is free, because the config volume already persists
transcripts, and the browser would share history with the terminal. But a
permission request answered in the browser is not an entry in a CLI transcript,
so prompts, their answers and our own error events would be lost or
reconstructed on replay, and the transcript format is not a stable interface.

## Decision

SSE with one input endpoint, and an application-owned event log as the record.

The session model already says the log is the truth rather than the connection,
and `Last-Event-ID` makes that literal. The argument for WebSocket rests on the
permission answer needing a channel, which it does not: a POST resolves the
pending promise just as well.

Token-level deltas are the one thing outside the record. They are sent live
without a sequence number and never replayed; a returning viewer sees the
finished message instead. A client that ignores deltas entirely still renders a
correct transcript.

## Consequences

The client writes one renderer for live and replayed events, and reconnect
handling is a header plus a slice.

A session run from the terminal in the same repository stays invisible to the
web UI, and the reverse. The two histories are separate records of separate
things.

We own the log's durability and its format. The format is `protocol/src/events.ts`,
which the client depends on directly, so changing an event shape is a contract
change and not an implementation detail.

Nothing may mount response compression in front of the stream, and the
disconnect path has to be proven on this proxy rather than assumed.
