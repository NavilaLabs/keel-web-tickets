# 0011. A workspace on the wire is a discriminated union, not a set of flags

Status: accepted
Date: 2026-09-20
Ticket: 7
Block: b2

## Context
`WorkspaceSummary` was `{ id, name, keel: boolean }`, hand-written in both the
client and the server and the only client-server crossing outside
`@keel-web/protocol`. Ticket 7 moves it into the protocol package, which makes
its shape a decision rather than an accident.

Two things forced the question. A workspace can now be listed while its path is
gone, so there is a second thing to say about it. And the existing `keel`
boolean was already wrong in a way nobody had noticed: the API derived it from
`tracker !== undefined`, while the session registry refused a workspace on
`ticketRepository === undefined`. A repository with a `ticket_repo` but no
`tickets` block showed as unusable although it was usable, and one with the
reverse showed as usable and then failed at attach.

Tickets 8 through 11 all consume this type: the phase display needs the ticket
repository, the artefact views need it too, the ticket and pull request views
need the tracker, and autocomplete needs the path.

## Options
### Two booleans plus reasons
`{ id, name, path, reachable, keel, reason? }`. The smallest change from what
exists. It permits combinations that cannot occur, such as an unreachable path
whose keel configuration is somehow known, and leaves it ambiguous which of the
two booleans a single `reason` belongs to. Two reason fields make that worse
rather than better.

### A status enum plus an optional reason
`status: 'ready' | 'unreachable' | 'unconfigured'` with `reason?`. This mirrors
`SessionFailureCode` plus `message`, which the protocol already uses, so it
introduces no new convention and costs consumers nothing. Its weakness is the
one the Kubernetes API conventions name against `phase` fields: `ticketRepository`
and `tracker` still hang beside the status, optional in every state, so the
inconsistency between the two definitions of usable survives the move.

### A discriminated union
Three cases, with `id`, `name` and `path` common to all. Only `ready` carries
`ticketRepository` and `tracker`. Illegal states cannot be constructed, a
`switch` is exhaustively checkable, and the two definitions of usable are
forced into one because there is exactly one place the ticket repository can
live. The cost is a narrowing step in every consumer, including one that only
wants to render a name.

### A conditions array
The Kubernetes answer once the number of independent facts grows. Overkill for
three cases, and it trades a compiler-checked shape for a runtime search.

## Decision
A discriminated union on `state`, with `ready` carrying the ticket repository
and the tracker.

The three cases are ordered by what can be known rather than by severity: an
unreachable path says nothing about the configuration inside it, so
`unreachable` wins over `unconfigured` instead of both being reported. This is
also why the original framing of the ticket, two independent states, is not
what got built. They are not independent, and modelling them as two flags would
have claimed a freedom that does not exist.

`reason` is one sentence written for the developer, not an error code. The
enum says what happened, the sentence says what about this particular
directory.

## Consequences
The mismatch between the sidebar and the session layer cannot come back,
because there is one condition and it is the presence of the ticket repository
in the `ready` branch.

Tickets 8 through 11 get a type that narrows to exactly what they need: the
phase display and the artefact views cannot reach for a ticket repository that
is not there, because in the other two branches it does not exist.

A fourth state later is a breaking change for every consumer, which is the
price of exhaustiveness. That is the right trade here because the consumers are
all in this repository. It would be the wrong trade for a published API.

Every consumer pays a narrowing step. The one that only renders a name pays it
for nothing, which is the honest cost of the choice.
