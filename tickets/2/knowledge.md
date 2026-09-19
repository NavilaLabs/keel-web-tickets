# 2: Linting, logging and tests

Source: https://github.com/NavilaLabs/keel-web/issues/2

## Goals
- [x] A developer can lint the whole project (client and server) with one command  `agreed`
- [x] A developer can format the whole project with one command, and formatting is checked automatically  `agreed`
- [x] Lint and format run automatically before each commit via prek  `agreed`
- [x] The server logs through a shared logger and logs requests by default; output is pretty in development and JSON in production  `agreed`
- [x] A test runner is set up for client and server with a root `test` script; no sample tests (the first ticket that needs one writes it)  `agreed`
- [x] A very basic CI runs lint, format check, build and tests on pull requests and on pushes to main  `agreed`
- [x] All tooling is installed in the Docker container and can be run from outside it (host) with docker or docker compose; the developer works on the host, not inside the container  `agreed`
- [x] The chosen tools work on Node, Vite 8, React 19 and TypeScript 6 (ADR 0001) so later tickets can use them without setup  `agreed`

## Problems
- Ticket has no acceptance criteria and no comments  `open`
- Ticket does not say which tools to use, or whether client, server or both are in scope  `resolved: client and server, tools chosen per block at step 6`
- Ticket 1 left out CI; #2 does not mention it  `resolved: basic CI included, confirmed by developer`
- Only the client has a linter (oxlint); the server has none, and there is no formatter, logger, test runner or root lint/test script  `open`
- Tooling must run in the container while the developer works on the host  `open`

## Open questions
- How is the container run from the host?  `answered: docker compose with a Dockerfile, run via docker compose exec (ADR 0003)`
- How does the prek hook (runs on the host at commit time) reach the tools in the container?  `answered: prek installed in the container, host hook script calls docker compose exec (ADR 0003)`
- Does the CI run through the same container image or natively on the runner?  `answered: natively on the runner (ADR 0003)`
- Which linter, formatter, logging library and test runner?  `answered: oxlint + oxfmt (pinned), Vitest with projects, pino with own middleware (ADR 0004)`
- Do #4 (Claude Code in the web UI) or its author have needs for logging or tests that should shape this ticket? Body of #4 was not read  `open`
- How the built client is served in production, and SSE vs WebSocket for streaming (carried over from ticket 1)  `open`

## Theme blocks
- **b1** Quality tooling: linting, formatting, prek pre-commit hook, test runner setup, basic CI, container execution, status `done`
- **b2** Server logging: logger and request logging, pretty in dev and JSON in production, status `done`

## Log
- 2026-09-19: Step 4: split into two blocks confirmed by developer. Developer added that work happens outside the container, so tooling is installed in the container and run from the host through docker or docker compose. This affects b1 (prek, CI, scripts) and adds the container-execution goal.
- 2026-09-19: Step 6 answered: docker compose plus Dockerfile, prek installed in the container and run via `docker compose exec`, format hook check-only, pino approach as recommended. Step 7: b2 claims keelWeb.server.logger and freezes server/src/logging/types.ts; b1 claims nothing (tooling is not modelled).
- 2026-09-19: Step 7 approved by developer. Stub comments and parameter names revised afterwards to the developer's comment rules (no signature or guarantee changed, fingerprint updated). Hook decision: prek runs only in the container, the host hook script aborts with a clear message when it is stopped.
- 2026-09-19: The sequential-implementation hook counted the finished b1 at step 9 as active and blocked b2. Fixed in the keel repo (98f9299, is_implementing ignores done blocks), b2 then implemented. Both blocks verified: lint, format check, build and prek pass, the server logs JSON in production and pretty output otherwise, the stub fingerprint is unchanged.
- 2026-09-19: PR #5 review comment: the doc comments in server/src/logging/types.ts can be shorter or omitted. Jump from 11 to 7 for b2 (pr_feedback), because the comments carry the frozen contract's guarantees. Consultation c7 decides which stay.
- 2026-09-19: c7 answered: the doc comments of types.ts were shortened to the guarantees a caller cannot read from the types. No signature changed. Fingerprint updated, b2 verified and done again.
