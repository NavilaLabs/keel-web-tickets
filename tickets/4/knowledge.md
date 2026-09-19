# 4: Integrate claude code into the web ui

The ticket body is empty. Title, ticket 1's v1 goal list and ADRs 0001 to 0004
are the whole input; everything below was reconstructed at step 2 and agreed
with the developer there.

## Goals

- [x] The developer chats with Claude Code from the browser: a message goes out, the
  answer streams back token by token, and the conversation keeps its history.  `agreed`
- [x] The session is a real Claude Code session with its tools, skills and MCP servers,
  so the keel workflow itself is drivable from the browser.  `agreed`
- [x] The developer sees what the agent is doing, not only its prose: tool calls, the
  files touched, and above all permission requests, answerable in the browser.  `agreed`
- [x] A session survives a page reload. Sessions are keyed by ticket id; a reload
  reattaches to a live run and replays the transcript otherwise.  `agreed`
- [x] Mutation stays with Claude Code. The UI never writes tickets, pull requests or
  diagrams itself, per ticket 1. This ticket must not open a second write path.  `agreed`
- [x] Server-side work is observable through the `Logger` frozen in ticket 2. ADR 0004
  named this ticket as that contract's first consumer.  `agreed`
- [x] Scope is the chat only. The ticket, pull request and LikeC4 view panes stay for a
  later ticket, but the layout is designed so they fit without a rebuild.  `agreed`

## Problems

- The ticket is empty: no acceptance criteria, no comments, no definition of done.
  `resolved: goals reconstructed at step 2 and confirmed, open points asked at step 3`
- Streaming transport was never decided, open since ticket 1 (SSE against WebSocket).
  Goal 3 presses on it, because permission prompts need a client to server direction
  mid-stream.  `open: decided in b1 step 6`
- The session model did not exist.  `resolved: one session per ticket, resumable;
  a server restart loses the live run but not the history`
- Working directory and tool permissions are a security decision, not a default: a
  browser that can run Claude Code unrestricted is remote code execution against the
  container, and both repos are mounted.  `resolved: the agent runs in the code repo
  exactly as the terminal does, reaching the ticket repo through the KEEL_TICKET_REPO
  mount, and every permission request is answered in the browser`
- Authentication for Claude Code is container-local and interactive (ADR 0002). The
  server can only run the SDK after the one-time `claude` login inside the container,
  and there is no API key path.  `open: behaviour when the login is missing is part of
  b1`
- The logging contract has two known soft spots here (ADR 0004): request duration is
  measured to the end of `next()` rather than the end of the response body, which is
  wrong for a stream, and code outside a Hono request has no logger until
  AsyncLocalStorage is added, which is exactly what a long-lived session is.
  `open: addressed in b1`
- The client is still the unmodified Vite starter: no router, no component structure,
  no styling library, and the dev proxy has no streaming-specific buffering or timeout
  configuration.  `open: addressed in b2`
- No tests exist yet. Ticket 2 set the rule that the first ticket needing one writes
  it, and this is that ticket.  `open`
- The as-is model lags the code by one ticket: ticket 2's `keelWeb.server.logger`
  component is merged in code but lives only on branch `ticket/2` in the ticket repo,
  so `main` has no component level for the server at all.  `open: the to-be branch
  ticket/4 is cut from main 9dd0982, re-check at 7.0`

## Open questions

- Which streaming transport, and how the permission answer travels back.  `open`
- How a missing Claude Code login inside the container surfaces in the browser.  `open`
- How the built client is served in production, carried over from ticket 1.  `open`
- Whether the UI stays localhost-only. No authentication is in front of it, which is
  fine for a single developer on the host but must not be exposed by accident.
  `answered: localhost only, no auth for now; recorded here as a standing risk`
- Whether session history is read from Claude Code's own session storage or from
  storage this application owns.  `open: part of b1`
- Relation to the keel project itself, whose workflow this UI is meant to drive. That
  repository was not examined and ticket 4 does not reference it.  `open`

## Theme blocks

- **b1** Server: run Claude Code sessions through the Agent SDK and expose them over a
  streaming API. Session registry keyed by ticket, lifecycle and resume, fixed working
  directory, permission gate, logging, endpoints and the event envelope.  `pending`
- **b2** Client: app shell (sidebar, main area, chat column) and the chat surface,
  implemented against the wire types b1 freezes at 7.2.  `pending`

The cut is the wire protocol. The dependency runs one way, b2 consumes what b1 freezes,
and the claimed components do not overlap: server against client.

## Log

- 2026-09-19 - Intake. Ticket body empty, so step 2 reconstructed the goals from ticket 1
  and the ADRs; the developer confirmed them and set the scope to chat only, with the
  layout anticipating the later view panes. Step 3 settled sessions per ticket,
  the agent running in the code repo with permission prompts answered in the browser,
  a sidebar plus main plus chat column shell, and localhost-only access without
  authentication.
