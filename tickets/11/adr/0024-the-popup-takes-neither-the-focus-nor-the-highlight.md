# 0024. The completion popup takes neither the focus nor the highlight

Status: accepted
Date: 2026-09-20
Ticket: 11
Block: b2

## Context

The completion popup is the part of this ticket that decides whether the web
chat feels like Claude Code, because it is the part the developer's hands
touch. Two choices in it are not obvious, and both are ones a reader would
later wonder about.

The first is what owns the focus. The input is a textarea, and the popup has
to appear beside it while the developer keeps typing into it. The component
libraries available here are built the other way round: cmdk owns an input of
its own and filters inside it, and Base UI's combobox is designed around the
input it controls.

The second is whether a row is highlighted when the list opens. In most web
interfaces the first row is, and enter takes it. In Claude Code's fullscreen
renderer none is, and enter sends the prompt as it was typed.

## Options

### Focus: a component library owns the input

Positioning, portalling, the ARIA roles and the keyboard come for free. It
means either giving up the textarea, which is what makes a multi-line message
possible, or fighting the library to let an outside element keep the focus.
Both are more work than they look, and the second leaves the behaviour
depending on internals no contract covers.

### Focus: a listbox that never takes it

A positioned list with `role="listbox"`, driven from the textarea's own key
handler, with `aria-activedescendant` naming the highlighted row. Roughly two
hundred lines including its tests, and every key is decided in one place
rather than split between the input and a library. The accessibility is then
keel-web's to get right rather than to inherit.

### Highlight: the first row, as on the web

Enter takes the top match. It is what a web interface usually does, and a
developer who has just typed `@ser` and sees `server/src` highlighted can
press enter once.

It also means enter no longer sends. A developer who typed a message ending
in a word that happens to open a popup would send a completion instead of
their message, and the failure is silent: the message goes, just not the one
they wrote.

### Highlight: none, as in the terminal

Enter always sends what is written. Taking a completion is tab, or an arrow
key first and then enter. Nothing the developer did not choose can end up in
the message.

It costs one keystroke in the common case, and it is not what a web interface
usually does, so it has to be learned once.

## Decision

The popup is a listbox that never takes the focus, and it opens with no row
highlighted. Enter sends. Tab takes the top row, or the highlighted one once
an arrow key has chosen it. Escape closes the popup and leaves the text
alone. The mouse hovers to highlight and clicks to take, so neither hand has
to move to the other.

This is the terminal's behaviour, and it was the developer's call between the
two.

## Consequences

Enter means one thing everywhere in the composer, which is the property worth
having: no popup can change what sending does. The cost is a keystroke and
one thing to learn.

keel-web owns the popup's keyboard and its accessibility. There is no library
to upgrade and none to work around, and the same listbox serves both the
slash and the file triggers, so they cannot drift apart.

Up is overloaded: it walks the history when the popup is closed and the caret
is in the first line, and moves the caret otherwise. That is the terminal's
rule and it is the one place in the composer where what a key does depends on
where the caret is.

Neither choice here is reversible for free. Moving to a component library
later would mean rewriting the key handling, and changing the highlight would
change what enter does, which is the thing a developer's hands learn first.
