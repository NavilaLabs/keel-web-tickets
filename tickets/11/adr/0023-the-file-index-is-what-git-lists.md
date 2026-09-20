# 0023. The file index is what git lists, rebuilt when the popup opens

Status: accepted
Date: 2026-09-20
Ticket: 11
Block: b2

## Context

Typing `@` has to offer the files and directories of the workspace. The agent
gives keel-web nothing for this: its `fileSuggestion` and `respectGitignore`
settings steer the terminal's own picker and never reach a browser. So the
server has to produce the list, and it is the only component that can, since
it runs on the developer's machine.

What exists today does not help. The directory browser lists subdirectories
only and skips anything starting with a dot, because it was written so a
repository can be picked rather than typed.

Two things have to be settled together: where the list comes from, and how
fresh it is. They are one decision because the cheapest source is also the
one that is expensive enough to need caching. A repository of any size is
tens of megabytes of paths, and a completion runs on every keystroke.

## Options

### `git ls-files --cached --others --exclude-standard`

One process, and git already knows the answer: the tracked files plus the
untracked ones that no ignore rule covers. `.gitignore`, `.git/info/exclude`
and the developer's global excludes are all honoured without keel-web
knowing they exist. Directories come from the path prefixes of that list, so
a directory appears exactly when it holds something worth naming.

It is also what the terminal does, so the completions match what the same
developer sees there, including where the terminal falls short: a nested
repository or a submodule is invisible, and a file that is still tracked but
deleted is listed.

It costs a hard dependency on git being runnable, and it has nothing to say
about a workspace that is not a git repository.

### Walking the filesystem, with an ignore library

Works without git, and works in any directory. It means reimplementing what
`.gitignore` means: nested files, negations, the global excludes. That is a
well-known source of quiet wrongness, and the failure mode is the worst one
available here, which is offering the developer a file the agent will refuse
to read or, worse, one that should never have been listed.

### `rg --files`

Fast, ignore-aware, and it works outside a git repository. It finds the
nested repositories the terminal's own picker misses. It adds a binary
keel-web does not depend on today and cannot assume is installed.

### Freshness: a filesystem watcher

Keeps the index warm, so a file the agent just wrote is there at once. It
costs a watcher per workspace whether or not anyone ever types `@`, and on
Linux a recursive watch over a large tree runs into the inotify limits. The
ignore files would have to be watched as well.

### Freshness: rebuilt when the popup opens, past a short age

No watcher, and nothing runs while nobody is completing. The kept index is
answered from immediately and a stale one is replaced behind the answer, so
the keystroke never waits for git. A file the agent wrote a moment ago can
be missing from one answer and present from the next.

## Decision

Git, with directories derived from the file list, cached per workspace and
rebuilt when the popup opens on an index older than a few seconds. The kept
index answers meanwhile.

A workspace with no git repository gets an empty list and a `reason` sentence
rather than a list built another way. That is what the terminal does, and a
half-right list is worse than an honest nothing: the developer can see why
there is no completion instead of wondering why their file is missing.

Git runs with `GIT_OPTIONAL_LOCKS=0`, because the agent is running git in the
same repository at the same time and a completion has no business taking the
index lock.

Filtering happens on the server and the answer is capped, so the repository
never travels to the browser.

## Consequences

The completions match the terminal's, which is what the ticket is for, and
they inherit its gaps: a submodule or a nested repository cannot be named by
completing, only by typing the path out.

keel-web now needs git on the PATH for this one feature. Everything else
keeps working without it, and a workspace where git cannot run says so in the
same `reason` a non-git workspace uses.

The index is a second thing that can be stale, bounded by its age rather than
by an event. The one case that matters, a file the agent just created, is
covered by the rebuild being tied to opening the popup rather than to a
timer: the developer who wants to reference that file opens the popup to do
it.

Nothing here handles a repository so large that the listing itself is slow.
The answer is capped and the build is shared between concurrent callers, but
a first `@` in a very large repository will wait for git once.
