# 0021. What a session runs with travels as state, not as history

Status: accepted
Date: 2026-09-20
Ticket: 11
Block: b1

## Context

The browser needs four things it has never been told: the commands a slash
completion can offer, the models a session can be switched to, and the model,
mode and effort it is running with now. The command list also changes while a
session runs, because the agent discovers skills as it works, and the agent
announces that by pushing the whole list again.

ADR 0005 gave keel-web one stream per session with an application-owned event
log behind it. Every event on it carries a sequence number, is appended to the
ticket repository, and is replayed unchanged to a viewer that reconnects.
Deltas are the one exception: they carry no sequence number, are never
recorded, and a reconnecting viewer sees the finished message instead.

Whether this new information is an event or an exception decides what a
transcript contains for the life of the project, and block b2 builds its
completion popup on whichever it is.

## Options

### A recorded event, like every other

`session.controls` joins `ServerEventBody`, gets a sequence number and is
appended. Replay is free: a viewer that reconnects is told everything it
missed without the server doing anything special.

It also writes the full command list into the transcript on every change, and
the agent pushes that list again whenever it discovers a skill. A ticket's
record would fill with copies of a list nobody will ever read back, and every
reader of a transcript would have to know to skip them. It would make the
record of what happened contain things that never happened.

### An unrecorded live message, like a delta

It travels on the same stream with no sequence number, is never appended, and
the server sends the current one to each viewer as it attaches.

The transcript keeps holding only what happened. The cost is that the server
has to send it on attach, because a live-only message is never replayed, and
a viewer that connects during a quiet session would otherwise have nothing.

### A separate endpoint the browser polls or fetches

A `GET` next to the stream, re-fetched when the browser thinks it might have
changed. It keeps the stream untouched and needs no new message type.

It gives the browser no way to learn about a change the agent made by itself,
which is exactly the case the command list exists for, and it adds a second
thing that can be stale.

## Decision

An unrecorded live message on the existing stream, sent once when a viewer
attaches and again on every change.

Each message carries the whole state rather than a delta, so a client that
has seen only the latest one is fully up to date and a lost message costs
nothing. This is also what the agent itself does with the command list.

## Consequences

The transcript keeps its meaning: it is what happened, and replaying it
twice still produces the same chat. A reader of the event log does not have
to know about models or commands at all.

The server carries one more duty at attach time. A viewer that misses the
message because it arrived before the listener was ready would show no model
and no commands until the next change, so the attach path has to send it
after the viewer is subscribed, not before.

Block b2's popup reads the command list from this message and never fetches
it. If a later ticket wants the list before a session exists, this gives it
nothing, because there is no stream until there is a session.

The client has one more thing that is not in the transcript store. What the
session runs with is held next to it rather than inside it, fed from the one
stream the transcript store already owns, so no second connection is opened
for it.
