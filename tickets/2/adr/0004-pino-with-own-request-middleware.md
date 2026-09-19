# 0004. Server logging with pino behind a narrow `Logger` interface and an own Hono request middleware

Status: accepted
Date: 2026-09-19
Ticket: 2
Block: b2

## Context
Every server code path will log, and #4 (Claude Code integration) is the first consumer, so the logging interface is something others build against. Requirements: shared logger, request logging by default, pretty in development, JSON in production, level from env, readable through `docker compose logs`.

## Options
### hono-pino
Least code, but a pre-1.0 community package whose README reportedly points to a successor; header logging unverified.
### Own middleware on a shared pino instance with `hono/request-id`
About 25 lines owned by us, no dependency risk, exact control over logged fields and redaction.
### `hono/logger`
Built in, but plain text lines, no JSON fields.
### Other libraries (consola, winston, LogTape)
Not clearly better for a JSON-plus-pretty split; pino was the developer's preference.

## Decision
pino, wrapped by a narrow `Logger` interface so callers do not depend on pino, plus an own request middleware with `hono/request-id`. Pretty output through an in-process `pino-pretty` transport selected by `NODE_ENV`; fallback is the synchronous pretty stream, then piping. Context via `c.get('logger')` only; AsyncLocalStorage may be added later without changing the contract. Headers, query and bodies are never logged, with a `redact` list as backstop.

## Consequences
Easy: swapping the wiring or the logger later, stable field names for log parsing. Hard: we own the middleware edge cases (status on thrown errors, streaming durations). Not available to code without a Hono context until ALS is added. `NODE_ENV=production` must be set in production. Risk: pino's typed overloads may not be structurally assignable to the narrow `Logger`; if so the contract needs a small change (a jump to 7).
