# 0017. An artefact is addressed the way keel points at it

Status: accepted
Date: 2026-09-20
Ticket: 9
Block: b1

## Context

Four things have to agree on what identifies one artefact: the tree the developer clicks, the
tabs that remember what is open, the URL that survives a reload, and the hint keel emits when a
consultation opens. Disagreement between any two of them shows up as an artefact opening twice,
or a hint landing on nothing.

keel already fixed its half. Since keel 0.3.0 the `artifact_hint` line on `events.jsonl` carries
resolved targets rather than pointers into `state.json`:

```json
{"kind": "stub", "repo": "code", "path": "server/src/logging/types.ts", "symbol": "Logger"}
{"kind": "c4_view", "view": "server", "branch": "ticket/9"}
```

`state.json` cannot be that identity by itself. Its `artifacts.adrs` list crosses tickets, its
`c4_views` holds view ids with no branch, and stubs are not in `artifacts` at all but in each
block's `claimed_stubs`, relative to the code repository rather than the ticket repository.

## Options

### Mirror the hint's shape

`ArtifactRef` carries the same fields the hint's targets carry: a kind, the repository a path is
relative to, the path, a symbol for a stub, and a view with its branch for a diagram. Mapping a
hint target to a reference is then a rename, not a translation.

Cost: the shape is inherited rather than designed, including the split between a file and a view
that is not a file. A union with two path-carrying variants and one that carries none is less
tidy than a single record.

### An opaque identifier minted by the server

The tree hands out ids, and everything else passes them back. Tidy, and it hides the filesystem
from the client entirely.

Cost: an id means nothing in a URL, and it cannot be minted from a hint without asking the server
to resolve it first. A hint would then need a round trip before the centre could act on it, for a
signal whose entire point is immediacy. It also makes a deep link meaningless across restarts
unless the ids are stable, which means deriving them from paths again.

### A path and nothing else

One string per artefact. Simplest to pass around.

Cost: it cannot express a view, which is a name inside a model on a branch rather than a file,
and it cannot say which of the two repositories a path belongs to. Both would come back as
prefixes inside the string, which is a union with the type removed.

## Decision

`ArtifactRef` mirrors the hint's targets. `kind` names what the artefact is and therefore how it
is rendered; a file variant carries `repository` and `path`; a stub additionally carries the
`symbol` that `state.json` recorded; a view carries `view` and an optional `branch`.

Two kinds exist that keel never hints at, `document` and `c4Source`, because the developer
browses the whole ticket directory and not only what `state.json` names.

The URL carries the reference as query parameters rather than path segments: an artefact path has
slashes of its own, and the path grammar stays free for the ticket and pull request views that
tickets #8 and #10 will add.

## Consequences

b2 maps a hint onto an open call with no resolution step and no round trip, which is what lets a
hint act at the moment it arrives.

Equality is field by field, so a reference that names an already open artefact activates its tab
instead of opening a second one. Anything that constructs a reference has to construct it the
same way, which is why the kinds are a closed union rather than a free string.

Following keel's shape means keel's shape can change under us. It is a contract on a file
keel-web does not own, and a kind added there is a kind to add here. The alternative was a
translation layer that would have to change for the same reason, with an extra indirection to
keep in step.

The URL is uglier than a path would have been. A deep link into an artefact is something a
developer copies rather than types, so that is the cheaper side to pay on.
