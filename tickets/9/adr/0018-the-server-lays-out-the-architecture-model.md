# 0018. The server lays out the architecture model, the browser draws it

Status: accepted
Date: 2026-09-20
Ticket: 9
Block: b3

## Context

The architecture model is LikeC4 source in the ticket repository, and it is the one artefact that
is not a document. The developer asked for it to be as interactive as `likec4 serve`: zoom, click
an element, move between views, and follow the `link` attributes that point back into the code
repository.

Two facts shape the decision. First, `main` holds the as-is model while each ticket's to-be model
lives on that ticket's branch, and keel's own hint names a view together with the branch it
belongs to. What has to be rendered is therefore usually not what is in the working tree.

Second, ADR 0001 assumed "LikeC4's official Vite plugin and React components work as documented"
and nobody had checked. Checking it produced two corrections: there is no `@likec4/react` package
and no `@likec4/vite-plugin` package, both are subpaths of the single `likec4` package; and the
version fit is exact, `likec4@1.59.3` wanting React 19.2 and Vite 8, which is what this client
runs.

A spike settled what the research could not. `LikeC4.fromWorkspace` parses this model in 176 ms
and lays every view out in 103 ms with Graphviz as WebAssembly, needing no system package. The
laid-out model is reachable as `$data`, serialises to 59 kB of JSON, survives a round trip
including its `_stage` marker, and `createLikeC4Model` rebuilds it on the other side with every
view intact. `playwright`, which `likec4` depends on, has no install script in the version in
question, so it costs disk rather than a browser download.

## Options

### The server parses and lays out, the browser draws

The server reads the `.c4` sources of the requested branch, parses and lays them out, and sends
the model as data. The client renders it with likec4's React components.

Cost: likec4 on both sides. In the browser that means Mantine, XYFlow, Motion, XState and an icon
set, on the order of a megabyte gzipped, in a client whose dependency list is deliberately short.
The diagram renders inside a shadow DOM with its own styles, so nothing of that leaks into the
flat surface around it, and nothing of ours leaks in.

### The Vite plugin over a directory

The documented happy path, and the least code: point the plugin at a workspace and import the
generated module.

Cost: the workspace is fixed when the dev server starts, so a branch becomes a directory to
rewrite behind the plugin's back, and a production build freezes the model at build time
entirely. It also moves a concern the server owns, which repository holds what, into the client's
build.

### A self-contained build per branch, in an iframe

`likec4 build --output-single-file` per branch state, served from our own origin and embedded.
The client bundle stays as it is and the interactivity is `likec4 serve`'s by construction.

Cost: an iframe is a wall. The tabs, the URL and the handling of `link` attributes all stop at
it, and following a link into the code repository is precisely one of the goals. It also means a
Vite build of seconds per branch state and a multi-megabyte artefact to cache and evict.

### A static image per view

Cheapest, and it was rejected at step 3 when interactivity was agreed.

## Decision

The server reads the sources of the requested branch, lays every view out with Graphviz as
WebAssembly, and serves the laid-out model as data. The client draws it with likec4's React
components and reports what the developer clicks rather than acting on it.

Sources come from the branch's committed tree, except when the ticket repository currently has
that branch checked out, in which case the working tree is read. That is what the developer sees,
and both halves handle unpushed commits correctly.

## Consequences

The branch becomes a parameter of a request rather than a property of the deployment, which is
what makes "the to-be model of this ticket" and "the as-is model on main" the same code path with
a different argument.

Navigation between views and the `link` attributes are reported to us, so a click inside a
diagram opens an artefact in the centre's own tabs and lands in the URL like everything else. No
other option could do this.

The client bundle grows by roughly an order of magnitude more than anything else in it. It is
loaded only when a diagram is opened, and that is the mitigation rather than a solution. If it
turns out to be felt, the iframe option is still there, and it is what we would fall back to.

One thing remains unverified and belongs to implementation: whether likec4 exposes a way to
render an element's links ourselves, or whether they have to be intercepted on the shadow root.
The goal survives either way, the second is grubbier.

keel-web now runs a parser and a layout engine in its server process. That is real weight for one
artefact kind, and the reason it is worth it is that the alternative was showing pictures of a
model the developer is being asked to approve.
