# 0019. The hint travels on its own stream, keyed by the reading position

Status: accepted
Date: 2026-09-20
Ticket: 9
Block: b2

## Context

keel emits an `artifact_hint` into `tickets/<id>/events.jsonl` when a consultation opens on an
artefact. The rules come with it and are not ours to change: it is fire and forget, a consumer
acts only on what was appended after it attached, there is no hint id, and a hint must never be
applied twice or the developer's view is yanked back.

keel-web has no watcher and no incremental reading anywhere, so both the following and the
carrying are new. The file is small by the producer's own commitment, a few hundred lines for a
finished ticket, and it is tracked by git, so a checkout can replace it with a different file of
a different length while we are reading it.

## Options

### The existing chat stream, as a new message beside `AssistantDelta`

Cheapest: a member of `StreamMessage` with no sequence number and no SSE id, exactly the
category `AssistantDelta` already occupies, plus ten lines of client wiring.

It fails on what the stream is. `GET .../events` calls `sessions.attach` as its first act, which
starts an agent, and it ends with `session.failed` when the agent cannot start, after which the
client closes it for good. So the hint would need an agent to exist, and would die precisely in
the state where the artefact views still work perfectly. A developer reading a ticket's artefacts
with nothing running would never be pointed anywhere, and keel's own README calls the terminal
with no UI the normal case.

### A stream of its own, scoped to the ticket

`GET /api/workspaces/:workspaceId/tickets/:ticketId/hints`, no attach, no agent, no session. The
SSE id of each message is the reading position in the file after that line, so `Last-Event-ID`
on a reconnect means "carry on from there", and a first connection without one means "from now
on". The two rules keel states fall out of the transport rather than being enforced on top of it.

The cost is a second `EventSource` per open ticket, against a per-browser limit of six
connections over HTTP/1.1, which both hops here are. Three tickets open at once is the ceiling.

### A polling endpoint with the cursor in the browser

The server holds nothing: the route reads from a given offset and answers. No watcher, no
subscription registry, no connection spent, and a hidden tab stops asking for free.

It costs a poll interval of latency, which for this signal is invisible, and it introduces a
second transport style in a project that already streams. That inconsistency would need
explaining every time someone reads the code.

## Decision

A ticket-scoped SSE stream, with the reading position as the SSE event id.

Two things sit on top of it. A freshness window, because a cursor alone cannot tell a two-second
reload from a laptop that slept for an hour: both come back with a valid offset, and only one of
them should be moved. And a visibility rule on the client, because a hint means look at this now,
and a tab nobody is looking at cannot look at anything.

The six-connection ceiling is accepted knowingly. If a developer ever has more than three tickets
open at once, the polling option is the way out, and the client seam is the same either way.

## Consequences

A hint reaches the browser with no agent involved, which is the case that matters: reading the
artefacts of a ticket is possible without a session, so being pointed at one must be too.

Resuming and reading on are the same mechanism, so there is no separate bookkeeping to get wrong
and nothing is ever delivered twice. The one-shot rule holds because of how the transport works
rather than because everyone remembered to honour it.

The stream is now a second thing that reads a file from the ticket repository as it changes. The
artefact views poll with an entity tag (ADR 0016) and this follows a file. Two mechanisms for two
different jobs is defensible, but a third would not be, and the next thing that needs to know
about a change should use one of these.

keel-web now depends on a detail of keel's output format that keel is free to change. The hint's
targets already mirror `ArtifactRef` (ADR 0017), so the coupling is narrow, and a kind we do not
know is ignored rather than breaking the stream.
