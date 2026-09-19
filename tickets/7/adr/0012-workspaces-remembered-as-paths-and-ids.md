# 0012. Workspaces are remembered as paths and ids in the developer's configuration directory

Status: accepted
Date: 2026-09-20
Ticket: 7
Block: b2

## Context
The workspace list came from `KEEL_WEB_WORKSPACES` at boot, so adding a
repository meant editing `compose.yaml` and restarting. Ticket 7 makes adding a
repository something the developer does by pointing at a path, which means the
list has to survive a restart.

Ticket 4 had already removed a `keel_web_data` volume and settled that keel-web
keeps no store of its own: the transcript lives in the ticket repository, beside
`knowledge.md`, and the session id is in the transcript's own events. Whatever
this ticket adds must not reintroduce a store.

The workspace id was `sha256(absolutePath).slice(0, 12)`, chosen so it would be
stable across restarts and renames. It is not stable across a move, which
contradicts the protocol's own doc comment: "a workspace can move on disk
without becoming a different workspace".

## Options
### Nothing new: keep deriving everything from the environment
No file, no store. Adding stays a restart away, which is the goal this ticket
exists to remove.

### Remember in the browser
`localStorage` in the client. keel-web gains no file at all, which is the
strictest reading of "no store of its own". A second browser or a second device
then sees none of the repositories that were added, and a list of paths on the
machine is not per-browser knowledge.

### Remember the whole workspace
Write name, tracker and ticket repository into the file along with the path.
One read at startup, no filesystem work per workspace. It makes the file a
second answer to a question `.claude/keel.json` already answers, and the two
drift the first time a repository's configuration changes.

### Remember only paths and ids
A file holding, per workspace, where it is and who it is. Everything else is
re-read from the repository on every start and on every read of the list. The
file cannot go stale about anything except which directories were added, which
is the one fact no repository can state about itself.

## Decision
A file of paths and ids, at `$XDG_CONFIG_HOME/keel-web/workspaces.json`, with
`~/.config` when that variable is unset, empty or relative, which is what the
specification prescribes rather than a second fallback of our own.

Ids are random and assigned when a workspace is added. Identity is remembered
rather than derived, so a repository that moves keeps its id and its URL, and
the protocol's doc comment becomes true. Two consequences follow and are
accepted: the same directory added twice after a removal is a new workspace,
and the same repository cloned on another machine is a different workspace
there.

The file is written to a temporary file in the same directory and renamed over
the old one, using Node's own `writeFile` with `flush` and `rename`. No
directory fsync, which is stricter than `write-file-atomic` and `atomically`
both manage, and no dependency for ten lines. A reader therefore never sees
half a list; the worst case is a lost last change after a power cut.

A file that exists but cannot be parsed stops the server, naming the file and
the parse error. The list holds paths the developer typed by hand, and quietly
setting them aside would look exactly like losing them.

The registry keeps a synchronous view in memory and refreshes it when the list
is read. Reachability is a fact about the world that changes without keel-web
being told, so it is displayed as of the last read rather than checked on every
access.

## Consequences
keel-web owns one file, and it holds the only thing that is genuinely ours:
which directories the developer chose. Everything else still comes from the
repository, so the rule from ticket 4 survives in substance rather than in
letter.

Two keel-web instances writing the same file are not coordinated with. The
rename keeps the file whole, so the failure is a lost addition, not a corrupt
list. This is documented rather than solved, because two instances need
different ports and the case is rare.

The registry contract changed in a way that inverts two of its old promises: a
missing path used to reject at startup and is now a listed state, and
`createWorkspaceRegistry` now takes a store rather than a list of paths. The
tests that encoded the old promises were rewritten rather than adapted.

Because the view is refreshed on read rather than on access, a path that
disappears between a read and an attach is found by the attach, not by the
sidebar. The session layer already refuses an unusable workspace, so the
outcome is a refused attach rather than an agent in the wrong place.
