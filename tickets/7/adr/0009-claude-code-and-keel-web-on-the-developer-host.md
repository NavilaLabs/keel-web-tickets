# 0009. Claude Code and keel-web run on the developer's host

Status: accepted
Date: 2026-09-20
Ticket: 7
Block: b1
Supersedes: 0002

## Context
keel-web is meant to be the working environment for the keel workflow on any
local repository. Until now the agent ran inside the devcontainer with a
Claude identity of its own (ADR 0002), and the one repository it could reach
was pinned by `KEEL_WEB_WORKSPACES` in `compose.yaml`.

Both of the things a workspace needs live on the host: the repositories the
developer works on, and the login they already have. A container identity
reproduces neither. It also cannot be handed to anyone else, because the
one-time login inside the container is a manual step per installation.

Ticket 7 has to decide where the agent runs before it can decide what a
workspace is, because a path only means something to a process that can see
it.

## Options
### Agent in the container, keep the named volume
No change. The container keeps its own login, and every repository the
developer wants to work on has to be mounted into it in advance. Cheapest,
and it is what exists. It forecloses the ticket's whole goal: a repository
that was not mounted at boot cannot be added at all.

### Agent on the host, server in the container
The agent moves to the host and the server stays containerised, reaching it
over a host-side helper. Keeps the container as the way keel-web is shipped.
The cost is that the agent is not the only thing that touches the
repositories: the workspace registry reads `.claude/keel.json` and the
transcript log writes `tickets/<id>/transcript.jsonl`, both from the server.
Each of them would need its own way across the boundary, and a path typed by
the developer would mean two different things on the two sides.

### Everything on the host
The server, the client's dev server and the agent all run on the developer's
machine, started from a clone with an npm script. One filesystem, one login,
one meaning for a path. The cost is that keel-web is no longer distributed as
a container that runs anywhere, and the reproducible environment the
devcontainer provided has to be kept for something else or given up.

## Decision
Everything on the host. The server is started from a clone with an npm script
and binds to `127.0.0.1`, and the agent is a child of that process, so it runs
as the developer, sees the same filesystem and resolves the same Claude Code
login.

The container survives for tests and CI only. It no longer maps ports, no
longer carries a Claude login, and the `claude_config` volume and the
devcontainer's Claude Code feature are removed.

Distributing keel-web as an image that anyone can start stays a goal for
later. It is not foreclosed: an image that reaches a host agent is a
different decision, and this one leaves it open by keeping no state that
would have to move.

## Consequences
A workspace can be any path on the machine, which is what makes ticket 7's
goal reachable at all. The developer's own login, settings, memory and history
are the ones the agent uses, so what keel-web runs is what `claude` in a
terminal would run.

Binding the server to `127.0.0.1` is now the access control. In the container
the port mapping did that job implicitly; on the host an agent with the
developer's credentials and their whole filesystem would otherwise be
reachable from the network.

The server process now outlives everything instead of being restarted with
the container, so anything it leaks per request compounds over a working day.
That is why the `stream detached` invariant became a test rather than a note.

The workspace ids in existing URLs change, because they are a hash of the
absolute path and the path is no longer `/workspaces/keel-web`. Block b2
detaches identity from the path, so this is a one-time break rather than a
recurring one.

Anyone debugging an environment-shaped problem no longer has a reproducible
environment to run the app in, only to run the tests in.
