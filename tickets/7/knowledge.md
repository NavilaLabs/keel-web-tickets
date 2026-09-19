# 7: Workspaces: add, select and switch between local repositories

keel-web can serve only one repository today, its own, because
`KEEL_WEB_WORKSPACES` in `compose.yaml` nails the agent's working directory
down at boot. This ticket makes a workspace something the developer adds,
picks and switches between, and moves Claude Code to the host where the
repositories and the login actually are.

Ticket 4 already delivered the workspace key itself: a session is a workspace
plus a ticket, the URL and the transcript path follow from it, and the wire
contract carries the dimension. Ticket 7 adds the choosing, the adding and
the persistence.

## Goals
- [x] Add a local repository by pointing at a path, from inside keel-web, without a restart. `.claude/keel.json` in the repository already says where its tickets live and which tracker they use, so there is no form to fill in.  `agreed`
- [x] Select a workspace and switch to another, and the chat, the tickets and the artefacts follow.  `agreed`
- [x] The list of added workspaces survives a restart without keel-web gaining a store of its own.  `agreed`
- [x] Claude Code runs on the host under the developer's own login, and keel-web runs beside it. This supersedes ADR 0002.  `agreed`
- [x] A repository without `.claude/keel.json` is listed, cannot hold a session, and says why.  `agreed`
- [x] A workspace whose path has disappeared stays listed, is marked unreachable and can be removed.  `agreed`

## Problems
- The workspace list is fixed at boot. `CreateWorkspaceRegistry` is frozen as `list()` and `find()`: no add, no remove, no reload, and the client fetches it once on mount. Changing it is a contract change to a ticket 4 stub.  `open`
- The agent belongs to the container. `assertLoggedIn()` probes `.credentials.json` under the container's config directory, `runQuery` runs in process, and two error messages say "inside the container" in so many words.  `resolved: the agent is a child of the server process on the host, the probe is gone (ADR 0010) and the configuration directory is handed to the agent instead of read.`
- The host move reaches past the agent. `createWorkspaceRegistry` reads `.claude/keel.json` and `createTranscriptLog` writes `tickets/<id>/transcript.jsonl` itself, both from the server. A host path only the agent can see would not carry them.  `resolved: keel-web itself moves to the host, so server, registry, transcript log and agent share one filesystem (c1).`
- `WorkspaceSummary` is hand-written on both sides and is the only client-server crossing outside `@keel-web/protocol`, which is why nobody noticed it.  `open`
- The keel capability flag contradicts itself. `GET /api/workspaces` reports `tracker !== undefined`, while the session registry refuses on `ticketRepository === undefined`. A repository can look usable and fail at attach, or look unusable while it is not.  `open`
- The workspace id is `sha256(absolutePath)`, so a moved repository becomes a different workspace, although the protocol's own doc comment claims a workspace can move on disk without becoming a different one.  `open`
- Exactly one `stream detached` line per browser reload has never been verified, and the host move changes the streaming topology. If it does not hold, every reload leaks a stream subscriber.  `resolved: it holds, and it is now a test against a real socket rather than a log to read. The in-process Hono harness never resumes the stream handler after an abort, so it cannot see this at all; on a real socket three reloads unsubscribed nine times before the route's detach was made idempotent.`
- ADR 0002 exists only on branch `ticket/1`, not on `main`. Marking it superseded means touching a file that `main` does not carry today.  `resolved: ADRs 0001 and 0002 were carried onto the ticket branch where the rest of the ADRs live, and 0002 is marked superseded by 0009 (c5).`

## Open questions
- How the keel-web process reaches Claude Code on the host.  `answered: it does not have to reach anywhere. With keel-web on the host the Agent SDK spawns the bundled CLI as a child process, under the same login and on the same filesystem (ADR 0009).`
- Where the keel-web process itself runs once the agent is on the host.  `answered: on the host alongside the agent; compose and the devcontainer stay as a development convenience (c1).`
- Whether distributing keel-web as an image anyone can start is part of this ticket.  `answered: a constraint to keep open, not a deliverable of ticket 7 (c1).`
- Where the list of added workspaces is kept, and whether it is per browser or per installation.  `answered: one small file in the user's config directory holding paths and ids, per installation. Name, tracker and ticket repository are re-read from the repository on every start, so it stays a list of pointers and not a second source of truth.`
- What happens to a workspace whose path has disappeared.  `answered: it stays listed, is marked unreachable, cannot hold a session and can be removed. An unmounted drive deletes nothing.`
- Whether a repository without `.claude/keel.json` can be added at all, and what it can do.  `answered: yes, listed but unable to hold a session, and it becomes usable as soon as the file appears. Keeps ticket 4's c15 answer and resolves the tracker/ticketRepository contradiction along the way.`
- Whether a moved repository stays the same workspace.  `answered: yes. The id is detached from the path hash and becomes a stable entry of the stored list, so a moved repository keeps its URL.`

## Theme blocks
- **b1** Claude Code on the host, keel-web beside it - `done`
  Runtime placement, how keel-web reaches the agent, the session registry
  without its container assumptions, an ADR for the arrangement and ADR 0002
  marked superseded, and the `stream detached` verification under the new
  topology.
  Claims `keelWeb.server.sessions`, the context relation `keelWeb -> claudeCode`
  and the `keelWeb.server` container itself.
- **b2** Workspaces the developer adds, selects and keeps - `pending`
  A mutable registry, the stored list of paths, identity detached from the
  path, reachability and usability as two separate states, `WorkspaceSummary`
  moved into the protocol package, and the sidebar that adds, selects and
  removes.
  Claims `keelWeb.server.workspaces`, `keelWeb.server.app`,
  `keelWeb.client.workspaceList`, `keelWeb.client.shell` and the crossing
  `client.workspaceList -> server.app`.

b1 runs first: the host move shifts the ground everything else stands on, and
verifying b2 in the old runtime would prove nothing once that ground changes.

The one seam between them is the definition of "usable". It belongs to b2,
whose registry owns it, and b1 only asks. Should step 7 show that the session
registry's signature has to move with it, the blocks were not independent and
that is a jump back to 4.

## Log
- 2026-09-19 - Step 2: confirmed that keel-web moves to the host with the agent, and that image distribution is a constraint rather than a deliverable (c1).
- 2026-09-20 - Step 4: split into b1 and b2, b1 first (c2).
- 2026-09-20 - Block b1, jump 8 -> 7 (`contract_change`): the protocol's `auth_required`
  failure code still said the container has no login. With no pre-flight probe it means the
  agent gave up before it was ready, and any early failure lands under it. `protocol/src/events.ts`
  was not frozen at 7.2, so the correction is a step 7 decision, not an implementation edit.
- 2026-09-20 - Block b1 done at step 9. 114 tests pass, lint, format and build are clean. Verified on a real socket: three reloads produce three attaches, three detaches and no surviving subscriber.
