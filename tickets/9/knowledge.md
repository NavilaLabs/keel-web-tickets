# 9: Artefact views: knowledge.md, ADRs, stubs and LikeC4 in the centre

The centre area is the reason the project exists: keel shows what is due now, and the developer
clicks through every artefact themselves. This ticket fills it with the ticket's artefacts.

## Goals

- [x] Every artefact of the ticket is readable in the centre - `knowledge.md`, the ADRs, the
      frozen stubs, the LikeC4 views - each rendered as its kind deserves rather than as raw
      text  `agreed`
- [x] The developer reaches any artefact at any time through their own navigation, without
      asking the agent and without a matching workflow step running  `agreed`
- [x] The centre follows the conversation: when a consultation opens on an artefact, that
      artefact comes to the front once  `agreed`
- [x] The developer keeps the upper hand: the hint switches once, the previous tab stays beside
      it, and nothing pulls the view back afterwards  `agreed`
- [x] Everything works with no hint at all. Navigation carries the feature alone; the hint is an
      addition  `agreed`
- [x] The open artefact is in the URL, so a deep link reopens it, and the centre remembers per
      ticket what was open  `agreed`
- [x] Artefacts are read-only in the browser, and keel-web keeps no store of its own for them
      (ADR 0012)  `agreed`
- [x] A missing artefact is explained rather than shown as an empty pane: a workspace without a
      ticket repository, a ticket without `knowledge.md`, a branch that does not exist  `agreed`
- [x] The outer sidebar holding workspaces and tickets collapses once a ticket is selected, so
      the centre has room  `agreed`

Deferred by decision, not forgotten: the diff of the to-be model against `main`, which step 7 of
the keel workflow is about. It needs git reads across two branches and a diff presentation for
diagrams, and it blocks none of the goals above. Its own ticket.

## Problems

- keel-web reads nothing of the ticket repository today. The only file it touches there is
  `tickets/<id>/transcript.jsonl`, which it writes. No route serves artefact contents  `open`
- There is no watcher and no incremental reading anywhere in the server: no `fs.watch`, no
  chokidar, and `readEvents` in `create-transcript-log.ts` reads whole files. keel#1's consumer
  rule needs a byte offset: a consumer acts only on hints appended after it attached, and what it
  reads on attach is history  `open`
- The hint has no channel to the browser. The existing SSE stream belongs to a chat session
  (ADR 0005), and `protocol/src/events.ts` is a contract, so adding an event is a contract
  change  `open`
- LikeC4 is rendered nowhere in either repository: no dependency, no plugin, no build step.
  ADR 0001 assumed the React components would work but never used them  `open`
- A `c4_view` target names a branch, so the model to render is not the one in the working
  directory but the one on the to-be branch  `open`
- Stubs live in the code repository, not the ticket repository, and only in
  `blocks[].claimed_stubs`. The centre therefore reads from two repositories  `open`
- A stub's file drifts from its fingerprint as soon as step 8 fills it, so "the frozen stub" and
  "the file today" are two different things  `open`
- `artifacts.adrs` crosses tickets: ticket 7 lists an ADR from ticket 1. The current ticket's
  directory does not define the set  `open`
- Serving file contents over HTTP widens the surface ADR 0014 left open. `127.0.0.1` is the
  access control, DNS rebinding defeats it, and paths that come out of a state file must be
  closed against traversal  `open`
- Tickets #8 (status) and #10 (ticket and pull request views) target the same centre area and
  the same `CentreView` union. This ticket must leave room for them without building them  `open`

## Open questions

- Does an open view follow the file when it changes on disk?
  `answered: yes, the content updates, without bringing anything to the front and without losing
  the scroll position. Only what is open follows.`
- Does "the frozen stub" show today's file or the state at step 7.2?
  `answered: today's file, with a note in the view when the fingerprint in state.json no longer
  matches.`
- How far does clicking through reach: only what state.json names, or everything under the
  ticket and architecture directories?
  `answered at step 7: everything in the ticket's own directory, whatever kind of file it is,
  plus the .c4 sources and the stubs state.json records. No filtering by kind at all. A file keel
  was asked to leave for the developer must not be invisible because keel-web does not recognise
  it, and that outweighs a tidy list. state.json only decides order and emphasis.`
- What does "like in an editor" mean concretely?
  `answered: a tree in its own sidebar on the left of the centre, tabs along the top, exactly one
  artefact visible. No split view.`
- What happens visibly when a hint arrives while the developer is elsewhere?
  `answered: the artefact comes to the front hard, as a new tab, and the tab that was in front
  stays beside it. One click is back.`
- Is the LikeC4 diagram interactive?
  `answered: yes, as interactive as "likec4 serve" - zoom, click an element, jump between views,
  follow the link attributes into the code repository.`
- Does the centre remember per ticket what was open?
  `answered: yes, per ticket, and the active artefact is in the URL.`
- Do status and permission prompts belong in the centre too?
  `answered: not in this ticket. The permission prompt stays in the chat column (ADR 0006) and
  the workflow status is ticket #8.`
- Which package renders LikeC4, and does it render at build time or at runtime?
  `answered at step 7: the single "likec4" package, whose react and vite-plugin subpaths are what
  ADR 0001 mistook for packages of their own. It renders at runtime: the server parses and lays
  out, the browser draws, so a branch is a request parameter (ADR 0018).`
- Does the hint travel on the existing SSE stream or on a channel of its own?
  `answered at step 7: its own ticket-scoped stream, because the chat stream starts an agent on
  attach and dies with it, which is exactly the state where the artefact views still work
  (ADR 0019).`
- Does a route that serves file contents need a request gate against DNS rebinding, which
  ADR 0014 left open for the directory route?
  `answered at step 6: yes, a Host header guard in front of every route, which is what Vite,
  Jupyter, Storybook and the MCP specification all settled on (ADR 0015).`

## Theme blocks

- **b1** Artefacts served, and readable in the centre - `done`
- **b2** The workflow points the centre at an artefact - `done`
- **b3** LikeC4 views, rendered interactively - `done`

b2 runs after b1 rather than beside it: it needs b1's frozen contract for how an artefact is
addressed and opened. Its step 6 research is independent. b3 depends on neither.

## Log

- 2026-09-20 - Step 9, b2 done, and verified against a live stream rather than only in tests: a
  hint appended to a ticket's `events.jsonl` arrived in the browser's stream as `artifact.hint`
  with the reading position as its id, and a reconnect carrying that id delivered only what came
  after it. Two defects were found while implementing: a listener attaching before keel had ever
  written the file treated the file's first appearance as history, and a test asserted that a
  same-length replacement is detectable, which by offset alone it is not.

- 2026-09-20 - Intake. keel#1, the producer of the hint, turned out to be implemented and
  installed already (keel 0.3.0). The `artifact_hint` contract was verified against the installed
  `hooks/state_writer.py` rather than taken from the ticket text: targets are resolved, ordered,
  and the first one is primary.
- 2026-09-20 - Step 4. Cut into three blocks rather than one. A single block would put the file
  route, the tailer, a new protocol event, the tree and the tabs into one step 7 consultation,
  which is where the workflow's main safeguard sits. A cut by layer, server against client, was
  rejected because neither half is independently plannable.
- 2026-09-20 - Noted for step 7.1: if b1 and b3 end up claiming the same server component
  because the `.c4` files travel the same route as everything else, the two were never
  independent and b3 merges into b1.
- 2026-09-20 - Step 6. Three researches. Two findings changed the ground: keel's `artifact_hint`
  is already implemented and installed, verified against the hook rather than the ticket text;
  and ADR 0001's assumption about LikeC4 is half wrong, since `@likec4/react` and
  `@likec4/vite-plugin` do not exist as packages, both being subpaths of `likec4`. The versions
  fit exactly.
- 2026-09-20 - Step 7. A spike settled what research could not: `LikeC4.fromWorkspace` parses
  this model in 176 ms, lays it out in 103 ms with Graphviz as WebAssembly, and `$data`
  serialises to 59 kB that `createLikeC4Model` rebuilds intact. Playwright, feared as a
  several-hundred-megabyte install, has no install script in the version `likec4` depends on.
- 2026-09-20 - Jump from 8 back to 7.2, block b1, on the first attempt to implement the reader.
  `read` was to report whether a stub still matches the hash frozen at 7.2, but its signature
  carried no ticket, and that hash lives in a ticket's `state.json`. Searching for the path across
  tickets would have reported whichever ticket was read first, so the ticket is now a parameter.
  The reader's unused dependency on the workspace registry went with it, and the model edge that
  claimed the same thing.
- 2026-09-20 - Step 9, b3 done. The production build, not the tests, found two defects that
  every green check had missed: importing `shiki` rather than its fine-grained entry points
  pulled every grammar it knows into the bundle, and likec4 sat in the main chunk instead of
  being loaded when a diagram is opened. Fixed, and the main chunk fell from 2.7 MB to 505 kB.
  Worth remembering: a passing test suite says nothing about what ships.
- 2026-09-20 - Jump from 9 back to 7.1, block b1. Verification found the model claiming an edge
  the code does not have, `centre -> router`: the shell owns navigation and hands the centre a
  callback, which keeps navigation in one place and leaves the centre a view that reports. The
  edge was dropped rather than the code changed to match it.
- 2026-09-20 - Step 9, b1 done. Also worth recording: the formatter removed a blank line from a
  frozen contract file, so its fingerprint was renewed deliberately and in the open. A
  fingerprint quietly renewed is a drift display that has stopped meaning anything.
- 2026-09-20 - Step 7. The overlap predicted at step 4 did appear, in the contracts rather than
  in the model: one protocol file and one route carried both blocks. Resolved by splitting the
  protocol file and reducing b1's knowledge of the architecture to a narrow port that returns
  view names, so b1 stands alone while b3 does not exist yet.
