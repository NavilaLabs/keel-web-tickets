# 0003. Dev tooling lives in a docker compose service and is run from the host with `docker compose exec`

Status: accepted
Date: 2026-09-19
Ticket: 2
Block: b1

## Context
The developer works on the host, outside the container. Lint, format, tests and the prek pre-commit hook must run with tools installed in the container only, and be startable from the host. CI must run the same checks. The repo has a devcontainer (image `typescript-node:1-22-bookworm`, claude-code feature, `KEEL_TICKET_REPO` bind mount) but no compose file or Dockerfile.

## Options
### docker compose with a Dockerfile
A `compose.yaml` service plus a Dockerfile that installs prek and the tooling. `devcontainer.json` points at the service. Works standalone with `docker compose exec/run`. Costs a new file pair and moving the devcontainer definition into compose; devcontainer features stay a layer on top.
### devcontainer CLI
Reuses `devcontainer.json` unchanged, but adds a host-side npm dependency, needs `devcontainer up` first, and is awkward in CI.
### Plain `docker run` wrapper
No new formats, but duplicates mounts, env and ports in a script and drifts from the devcontainer.

## Decision
Docker compose with a Dockerfile. Tools (prek, oxlint, oxfmt, Vitest) are installed in the image or its `node_modules` volumes. The host runs everything as `docker compose exec <service> <cmd>`, including `prek`. `node_modules` (root, client, server) are named volumes so Linux binaries stay in the container. prek runs check-only. The CI runs natively on the runner (setup-node, npm cache), not through the container, because the devcontainer's `initializeCommand` and ticket-repo mount need dummy values there.

## Consequences
One definition for devcontainer and command-line use, no extra host tool. `exec` needs a running container, so commits fail with a clear message when it is down unless a wrapper starts it. prek installs its git hook into `.git/hooks`, which the host git executes: the hook file must call `docker compose exec` (a tracked hook script via `core.hooksPath`), not the container's `prek` path. Host editors see an empty `node_modules`, and dependency changes require `npm install` inside the container. Foreclosed: relying on host-installed tooling. Unverified before implementation: whether devcontainer `mounts` work with a compose service, host UID (1000?), Vitest 5 with Vite 8 on the image's Node 22 minor.
