# 0002. Claude Code config in the devcontainer is a named volume

Status: accepted
Date: 2026-09-19
Ticket: 1
Block: b1

## Context
The devcontainer should make Claude Code available inside the container. The developer first asked for the local Claude Code instance to be mounted in.

## Options
### Bind-mount the host config
Bind `~/.claude` and `~/.claude.json` from the host. The container shares the host identity, no login needed. Pitfalls: Docker creates a missing `~/.claude.json` as a directory, host and container UIDs must match, concurrent writes to `.claude.json` can collide, the host is Arch and the container likely Debian, and credentials are readable by everything in the container.
### Named volume (documented pattern)
Mount a named volume at the container's `~/.claude` and set `CLAUDE_CONFIG_DIR`. Documented by Anthropic. Needs a one-time login inside the container, and the OAuth callback may need the paste-code fallback.

## Decision
Named volume. The developer chose the documented pattern over host bind mounts after seeing the pitfalls. Claude Code itself is installed in the container through the official devcontainer feature, not mounted from the host.

## Consequences
The container has its own Claude identity and needs a one-time login. No host credentials are exposed to the container and no UID or concurrent-write issues arise. The host's Claude settings, memory and history are not shared with the container.
