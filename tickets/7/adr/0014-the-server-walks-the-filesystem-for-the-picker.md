# 0014. The server walks the filesystem so a repository can be picked

Status: accepted
Date: 2026-09-21
Ticket: 7
Block: b2

## Context
Adding a repository meant typing its absolute path into a text field. That
satisfies the ticket's wording, "adding a repository means pointing at a path,
not filling in a form", only in the letter: a free-text field is a form with
one field, and the developer said so.

The obvious answer is the one every desktop application has. The browser does
not have it. A directory picker yields a handle scoped to the chosen folder and
deliberately never the path, and a directory input yields names relative to the
chosen folder. Neither can produce `/home/steffenk/projects/keel-web`, which is
what the registry needs.

What changed the situation is ADR 0009: keel-web now runs on the same machine
as the developer, as the developer. The process that cannot see the filesystem
is talking to a process that can.

## Options
### The File System Access API
`showDirectoryPicker()` opens the real system dialog, which is exactly the
feel that was asked for. It returns a `FileSystemDirectoryHandle` and no path,
by design and not by oversight, so the registry would have to be rebuilt around
handles that cannot be persisted as text, cannot be handed to the agent as a
working directory, and do not exist outside Chromium.

### A directory input
`<input webkitdirectory>` gives entries with `webkitRelativePath`, rooted at the
chosen folder. The absolute path is not among them, and the browser reads every
file in the tree to produce the list, which for a repository means walking
`node_modules`.

### The server lists directories
A route that answers with the subdirectories of a path, and a dialog that walks
them. The server already runs where the repositories are, so nothing has to
cross a boundary that browsers were built to defend.

## Decision
The server lists directories, over `GET /api/directories`, and the client shows
them in a dialog with a path line at the top that can also be typed into.

Only subdirectories are listed, sorted by name without regard to case, with
dot-directories left out. The listing carries absolute paths for its entries, so
choosing one needs no path arithmetic in the browser and no second idea of what
a path is.

An unreadable entry inside a readable directory is skipped rather than failing
the listing, because one root-owned folder in a home directory should not hide
everything beside it.

## Consequences
Picking a repository now feels like picking a repository, and the path that
reaches `add` is the one the filesystem gave rather than one that was typed.
The route to add by path is unchanged, so the API still works without the UI.

keel-web can now be asked, over HTTP, what is on this filesystem. That is a
capability it did not have, and it sharpens the request-gate question that this
ticket deliberately left open. A cross-site `GET` cannot read the answer,
because no CORS headers are sent. DNS rebinding defeats that, and where it
previously exposed the workspace list and the ability to add one, it now also
exposes the shape of the developer's filesystem. The gap did not change; what
is behind it did.

The listing is a read with no effect, so it needs no confirmation, can be
repeated, and can be abandoned mid-flight. That is why it is a separate
component from the workspace registry, which remembers things.
