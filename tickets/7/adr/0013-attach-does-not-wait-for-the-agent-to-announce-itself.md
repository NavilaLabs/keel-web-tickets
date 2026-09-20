# 0013. attach does not wait for the agent to announce itself

Status: accepted
Date: 2026-09-21
Ticket: 7
Block: b1
Amends: 0010

## Context
ADR 0010 removed the pre-flight login probe and said a missing login would
"reach the browser as a stream event rather than a rejected attach". It then
described the new signal as an agent that "exits before it reports itself
ready", and the implementation read ready as the Agent SDK's `system:init`
message, which `normalise` turns into `session.started`.

That reading is wrong, and running the chat in a browser against a real agent
is what showed it. `attach` waited for `session.started`, the SDK sends `init`
when the first turn begins, and a turn begins when a message arrives through
`send`. `send` refuses a key that has no session, and a session is only
registered once `attach` has resolved. The two waited on each other for sixty
seconds and then failed with `SessionStartError`, which is exactly what the
developer saw:

```
err: "Error: No session for 37ea77f0-.../7."          POST .../input  404
err: "SessionStartError: The agent did not report a session in time."
```

Measured against the installed SDK, holding the first message back by a fixed
delay:

| first message released at | `init` observed at |
| --- | --- |
| 15s | 15.2s |
| 10s, subprocess pre-warmed with `startup()` | 10.4s |
| immediately | 0.4s |

`init` tracks the first turn, not the process. The test suite never saw this
because its fake agent emits `init` as soon as it is asked for messages.

A second thing came out of reading the callers. `attach` is called in exactly
one place, `create-chat-routes.ts`, which discards the result. Nothing reads
`Session.sessionId`: the id used to resume comes from `transcript.lastSessionId`,
which reads the recorded `session.started` events. Nothing calls
`Session.pendingPermissions()` either, because a returning viewer gets held
requests from the transcript replay. The registry was waiting a minute to fill
a field nobody reads, inside an object its only caller throws away.

## Options
### Wait for nothing
Register the session as soon as the run exists and return. No deadlock is
possible and no timeout is needed. An agent that dies immediately is noticed
when the developer's first message produces a failure rather than at attach
time.

### Wait for `startup()`
The SDK exports `startup()`, which pre-warms the subprocess; measured, it
resolves in 0.2 to 0.3 seconds. `attach` would then promise something
checkable: the subprocess is up. It promises nothing about the login, and it
costs a rebuild of the injected `runQuery` seam that every session test hangs
on.

### Wait for the first sign of life, whatever it is
The first `system` message of any kind. In this repository those are the
SessionStart hooks at 0.3 seconds. In a repository without hooks none would
ever arrive, so the deadlock would only be harder to find.

## Decision
`attach` waits for nothing. It makes sure a session exists, and returns once
that session will accept messages.

Its return type becomes `Promise<void>`, and the `Session` interface goes with
it, along with `sessionId` and `pendingPermissions`. Keeping an object that
nothing reads would invite someone to read the one field that is empty until a
turn has run.

`SessionStartError` and `startTimeoutMs` are removed. Without a wait there is
no moment at which either could arise, and an error type that can never be
thrown is worse than no error type.

The only rejection left is `UnknownWorkspaceError`. Everything else, a missing
login included, arrives as `session.failed` on the stream, which is what ADR
0010 intended and what the client already handles.

## Consequences
The contract now states a guarantee that holds: after `attach`, messages are
accepted. What it no longer claims is that the agent is alive and authenticated,
because that cannot be known before the agent has been given something to do.

A broken agent is found one step later, when the developer's first message
comes back as a failure instead of an answer. That is the honest cost of the
change, and it is the same cost ADR 0010 already accepted in writing.

The session registry's tests were testing the waiting, so they were rewritten
rather than adapted. The fake agent that emits `init` on demand is the reason
this survived three consultations and a merge-ready pull request, so the tests
now also cover an agent that stays silent until it is spoken to.

Ticket 8 and after will want a session id for a session-scoped view. It is in
the transcript, where `lastSessionId` already reads it, rather than on an object
handed out by `attach`.
