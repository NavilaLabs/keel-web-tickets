# 0015. A host guard in front of every route

Status: accepted
Date: 2026-09-20
Ticket: 9
Block: b1

## Context

ADR 0009 made the loopback binding the access control, and ADR 0014 named the gap it leaves and
left it open: "keel-web can now be asked, over HTTP, what is on this filesystem. A cross-site
`GET` cannot read the answer, because no CORS headers are sent. DNS rebinding defeats that."

At the time the exposure was a list of directory names. This ticket turns it into the contents of
files in the developer's repositories, read and returned verbatim. The same gap now leaks the
work rather than the shape of the disk, so it is decided here rather than left open again.

The attack needs no privilege on this machine. A page the developer visits points a domain it
controls at 127.0.0.1, waits for the browser's DNS cache to expire, and then issues same-origin
requests to keel-web under that domain. The connection is local, so the binding sees nothing
wrong, and the response reaches the attacker's script because, to the browser, it is their own
origin.

## Options

### Check the Host header

Accept `127.0.0.1`, `[::1]`, `localhost` and `*.localhost`, refuse everything else with 403. A
rebound request carries the attacker's domain in `Host` and dies before it reaches a reader.

This is what every comparable local-first tool converged on: Vite (`server.allowedHosts`, added
in 5.4.12 and 6.0.9 with the commit message "check host header to prevent DNS rebinding
attacks"), Jupyter (since notebook 5.7, PR #3714), Storybook (PR #30523, defaulting to Vite's
behaviour), and the MCP specification for Streamable HTTP, which requires Origin validation and
recommends binding to 127.0.0.1.

Cost: a developer reaching keel-web through an alias or a tunnel gets a 403 and needs a way to
allow their own name.

### A token in the launch URL, the way Jupyter does it

The server mints a token at startup, prints it in the URL, and requires it on every request.
Strictly stronger: it survives rebinding, and it also survives another process on this machine
guessing the port.

Cost: it takes away exactly what ADR 0009 bought, which is that the developer opens
`http://127.0.0.1:3000` and is there. Tokens leak through `Referer` and shell history and need a
rotation story. For a single-user tool whose whole content is files the user already owns, the
bookkeeping outweighs what it adds over the Host check.

### Leave it, as ADR 0014 did

Defensible while the exposure was directory names. Not defensible once the contents of
`knowledge.md`, the ADRs and source files answer to a cross-origin page that only had to wait for
a DNS cache to expire.

### Rely on the browser

Chrome 142 shipped Local Network Access, which checks the resolved target address space and does
catch rebinding. It is browser-dependent, prompt-dependent and the user can allow it. Worth
knowing, not worth building on.

## Decision

A Host guard, mounted in front of every route: the request passes when `Host` names a loopback
address, and is refused with 403 and a sentence naming the host that was sent. `Origin` is
checked the other way round, refused when present and foreign, ignored when absent, because a
request from a terminal legitimately carries none. A missing `Host` is refused; HTTP/1.1 requires
it and a request without one is not worth guessing about.

The escape hatch is a list of additional host names, matched whole, for the developer who reaches
keel-web under a name of their own.

## Consequences

The gap ADR 0014 left open is closed, and this ADR is where that is written down, so the next
route that serves file contents does not have to rediscover the question.

A second access control now exists beside the loopback binding, and the obvious question is where
it ends. It ends here: the binding keeps non-browser callers out, the guard keeps the browser
from being used as a proxy into this machine, and there is no third threat that a third control
would answer.

The 403 will eventually confuse someone who put keel-web behind a name. The guard therefore says
which host it refused rather than failing blankly, and the option to allow a name exists before
anyone needs it.
