# 0016. An open artefact follows its file by asking again

Status: accepted
Date: 2026-09-20
Ticket: 9
Block: b1

## Context

An artefact on screen goes stale while the developer reads it. `knowledge.md` is rewritten on
every jump back, ADRs appear during step 7.3, and a stub grows an implementation in step 8. The
ticket requires that an open view follows its file without stealing focus and without losing the
reader's position.

keel-web has no watcher of any kind today: no `fs.watch`, no chokidar, and nothing in the
dependency tree that would bring one. Whatever this block picks is new machinery.

The decision is entangled with the block cut. Block b2 has to bring the workflow's
`artifact_hint` to the browser and will build a channel for it. If b1 builds a change channel
first, b2 becomes a thin consumer of b1's transport and the split into blocks was a split by
layer rather than by function.

## Options

### Conditional GET, the open artefact asks again

Every artefact answer carries an `ETag` derived from its content. The open view asks again every
few seconds with `If-None-Match` and is answered 304 with no body while nothing changed. Only the
artefact actually on screen polls, and only while the page is visible.

Cost: up to one interval of latency, and a request every few seconds over loopback for one small
file. Adds no dependency and no channel.

### A change channel, built here

A stream of change notifications, either as a long-poll that holds the request until the file
changes or as a dedicated SSE endpoint. Near-instant, and the same machinery b2 needs.

Cost: a server-side watcher, with the caveats that come with it. `fs.watch` on a file follows the
inode, so a git checkout or an editor's atomic save ends the watch silently, and watching the
parent directory instead means filtering events for a file that is rewritten alongside others.
More importantly it takes b2's subject away: the artefact hint would arrive on a channel b1
designed for a different purpose, and the two blocks stop being separable.

### Nothing, the developer reloads

Cheapest. Contradicts the agreed goal, and the case it fails is the normal one: reading
`knowledge.md` while the agent rewrites it.

## Decision

Conditional GET. The artefact routes answer with an `ETag`; the open view asks again on an
interval while the page is visible and replaces the content only when the answer is not a 304.

The change channel stays b2's to design, for the signal that actually needs one.

## Consequences

b1 adds no watcher, no dependency and no new wire event. The whole mechanism is an `ETag` header
and an interval, and it can be replaced by a push channel later without the client's model of the
world changing: it would still be told "this artefact changed", and still ask for it.

The cut into blocks holds. b2 owns the transport for the hint and can choose it on the hint's own
terms rather than inheriting one.

Replacing the content must not remount the scroll container, or the reader loses their place on
every rewrite. That is a constraint on the view, and it is written into the contract rather than
left to be discovered.

An artefact that changes twice within one interval is seen once, in its final state. For reading
a document that is correct rather than a compromise.
