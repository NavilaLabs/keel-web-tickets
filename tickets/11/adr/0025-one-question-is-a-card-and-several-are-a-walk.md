# 0025. One question is a card, several are a walk with a last look

Status: accepted
Date: 2026-09-20
Ticket: 11
Block: b3

## Context

Ticket 4 already renders what the agent asks through `AskUserQuestion`: every
question on one card, radio buttons or checkboxes, and a button to send. It
works, and the developer's complaint about it was short: they want to answer
with the mouse.

Reading that against what the tool actually sends explains the complaint. A
request carries one to four questions, each with two to four options, each
option with a description and sometimes a preview. All of that on one card is
a tall panel in a chat column, and the thing being clicked is a radio button
of a few pixels with text beside it rather than the option itself.

Two things were also simply missing. There is no way to answer anything other
than the options, and no way to decline. The tool's own schema says the
options must not include an "Other", because the client is expected to
provide it. And a question the developer does not want to answer can only be
escaped by stopping the turn, which throws away the work as well.

## Options

### One card, enlarged

Keep every question on one screen and make each option a full-width target
with its description inside it. Add a text field for a typed answer and a
button to decline.

The smallest change, and the developer sees every answer at once before
sending, so the review is free. Four questions with four options each, some
carrying previews, make a panel far taller than the column, and the preview
has nowhere to go: under every option it is unreadable, and beside them there
is no room.

### A walk through every question, always

One question per screen, a chip per question along the top to move between
them, and a last step listing the answers. The option list has room, so a
preview can sit beside it, which is what a preview is for.

It is the same shape whatever arrives, which is one code path. For the common
case, a single question with a single choice, it costs two clicks where one
would do: pick, then confirm, then it goes.

### One card for one question, a walk for several

A single question is one card, and clicking an option sends it. Several
questions walk, with the chips and the last look.

It fits both cases and costs two render paths to test. It is not one shape,
so a reader has to know both.

## Decision

The hybrid. A single-choice question on its own is a card that sends on the
click; anything else walks.

A question that allows several choices never sends on a click, whether it is
alone or not, because no single click can mean "and that is all of them". It
always ends in a step of its own.

"Other" is a text field beside the options rather than an option that opens
one, so it takes one click to reach rather than two. What is typed becomes
the answer's value, joined to any chosen labels by a comma and a space, which
is the one shape the agent reads.

Declining sends a `deny` with whatever the developer wrote. It does not carry
`interrupt`, so the turn continues and the agent decides what to do without
an answer. Previews stay as they arrive, in a monospace box, shown for the
option under the pointer.

## Consequences

The common case, which is one question, is one click from start to finish.
That is the case the developer was complaining about, and it is the one the
decision optimises.

Two render paths exist and both have to keep working. The one that sends on a
click is the one to be careful with, because it cannot be taken back: a
misclick is an answer. It is also what the developer asked for, and the
alternative was a confirmation step asking whether the last click was meant.

Nothing here changes the wire contract. A typed answer travels as the answer,
a decline travels as the deny that already existed, and the block is a client
change alone. `annotations`, which the agent would accept for a note beside a
choice, stays unused: nothing asked for it, and it would have to be carried
all the way through the protocol to be worth having.

The preview is rendered as the monospace block the installed types say the
agent emits by default. Asking the agent for HTML previews instead is a
setting keel-web never sends, and turning it on later would mean rendering
untrusted markup, which is a decision of its own.
