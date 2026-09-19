# 0010. The registry does not probe for a Claude Code login

Status: accepted
Date: 2026-09-20
Ticket: 7
Block: b1

## Context
`attach` refuses with `AuthRequiredError` when Claude Code has no login. It
decided that by checking for `.credentials.json` under the configured
directory, which was true enough inside a container where the only way in was
`claude` run once.

On the developer's host it is not. Claude Code accepts a login from a
credentials file, from `ANTHROPIC_API_KEY` or `ANTHROPIC_AUTH_TOKEN`, from an
`apiKeyHelper`, from `CLAUDE_CODE_OAUTH_TOKEN`, from an Anthropic profile,
from Bedrock, Vertex or Foundry, and on macOS from the Keychain. Every one of
those is a developer who is logged in and whom the probe would turn away. The
file can also be there while the login behind it has expired, which the probe
reads as success.

The registry's second guess is no better: it classifies a failed start as an
authentication problem by looking for `/login`, `invalid api key`, `not
logged in` or `unauthorized` in the thrown message. The message the SDK
actually throws is `Claude Code process exited with code N` plus a tail of
stderr, so those hits are luck rather than contract.

## Options
### Keep the file probe, pointed at the host's directory
One line changes. Right on the days the developer logged in with `/login`,
wrong for every other credential source, and unable to tell an expired login
from a missing one.

### Ask the CLI: `claude auth status --json`
The binary the SDK ships answers with `loggedIn`, `authMethod`, `apiProvider`
and the `configDirectory` it resolved, and exits non-zero when logged out.
A definitive answer before the session starts, and it says which credential
won, which is useful when the answer is surprising. The costs: a subprocess on
every attach, a JSON shape that is not published and so is coupled to the
CLI version, and resolving the binary's path without the resolver the SDK does
not export.

### Ask the SDK: `accountInfo()` or the init message
The session that will run is the one that answers, so the check cannot drift
from the thing it checks. `Query.accountInfo()` exists, and the `system:init`
message the registry already normalises carries `apiKeySource`. The check
moves after the subprocess is up, so a missing login is no longer a startup
rejection.

### No pre-flight at all
Start the session and classify what comes back. Correct for every credential
source by construction, because the thing that decides is the thing that
knows. The signal is the weakest of the four: an agent that exits before
reporting readiness has not authenticated, but it may also have failed for an
unrelated reason.

## Decision
No pre-flight. The registry starts the session, and an agent that exits before
reporting itself ready is reported as `AuthRequiredError`.

The imprecision is accepted rather than papered over: any early failure lands
in the same place, so the error carries the agent's own output with it, and
the developer reads the real cause instead of our guess at it. Substring
matching on the message is removed, because a guess that looks like a
diagnosis is worse than a diagnosis that admits its range.

`configDirectory` stays an input to the registry, but it stops being
something the registry reads. It is handed to the agent as part of its
environment, spread over a copy of the server's own, so that both sides
resolve the same directory instead of each reading `CLAUDE_CONFIG_DIR`
independently and happening to agree.

## Consequences
Every way of logging in to Claude Code works with keel-web, including the ones
nobody here uses yet, and none of them needed a case in our code.

The cost is a less precise message. An agent that fails to start for a reason
that has nothing to do with authentication is announced as an authentication
problem, with the real output attached. If that turns out to be confusing in
practice, the ADR to write is the one that adds `claude auth status --json`
as a second opinion after the failure, not one that brings the file probe
back.

A missing login is no longer known before the session starts, so the failure
reaches the browser as a stream event rather than a rejected attach. The
client already handles `auth_required` that way.

The tests lose the seam they used: they wrote a fake `.credentials.json` into
a temporary directory to make the registry believe in a login. They now drive
the injected `runQuery` instead, which is the seam that was there all along.
