# 0007. Build the client with Tailwind and shadcn/ui, flat by intent

Status: accepted
Date: 2026-09-19
Ticket: 4
Block: b2

## Context

The client is an unmodified Vite starter: no styling library, no component
library, no design. It has to become an app shell with a ticket list, an
artifact area and a permanent chat column, and it has to render two things
that are not ordinary UI: a tool call with arbitrary input, and a permission
prompt that must be answerable rather than merely visible.

`client/src/index.css` already carries a coherent token set with a
`prefers-color-scheme: dark` block, so dark mode is largely solved whichever
way this goes. What is not solved is everything else.

The decision is not only about styling. Every usable piece of prior art for
tool-call rendering and approval cards, assistant-ui, Vercel AI Elements and
shadcn/ui itself, is built on Tailwind. Choosing a styling approach therefore
also chooses whether those components can be adopted or have to be written.

## Options

### CSS Modules with the existing tokens

Zero setup: Vite handles `*.module.css` without configuration or packages, and
Vite 8's Lightning CSS gives nesting and prefixing for free. Nothing new enters
the container.

The cost is that the spacing and typography scale is invented here, and that
the tool-call view and the permission prompt are written from scratch. "Should
not look like a starter template" becomes a design problem with nothing to lean
on.

### Tailwind v4 with shadcn/ui

Two packages and one line in `client/vite.config.ts`; v4 needs no
`tailwind.config.js` and no PostCSS config, and theme tokens live in CSS via
`@theme`. shadcn additionally needs path aliases the project does not have yet.

It buys the option to adopt approval cards and tool views rather than write
them, and a component vocabulary that is consistent by construction. It costs a
large class vocabulary in every file, and `oxfmt` does not sort Tailwind
classes, so the usual Prettier plugin is not available here.

### A component library such as Radix Themes or Mantine

The fastest route away from a starter template, dark mode included, no Tailwind
required. It costs a large dependency whose opinions appear in every component,
and it does not open the door to the agent-specific prior art, which is where
the hard rendering problems are.

## Decision

Tailwind v4 with shadcn/ui, and React Bits where an accent genuinely helps.

The deciding factor is the permission prompt and the tool view, not the shell.
Those are the parts where prior art exists and where getting it wrong means a
dialog nobody can answer, so keeping the option to borrow is worth the
vocabulary.

The design is flat on purpose: the shadcn defaults are taken without their
shadow and radius flourishes, and the existing token set stays the source of
colour. React Bits is used sparingly, because animation and flatness pull
against each other.

## Consequences

Tailwind, `@tailwindcss/vite` and path aliases enter the client build, and the
starter `#root` rules in `App.css` go, since they fight a full-viewport
three-column layout.

Components arrive by being copied in, not installed, so they are ours to
maintain and to keep flat. That is the point of the registry model, and it also
means a shadcn update is never automatic.

Class sorting stays manual. If that becomes noise, it is an argument for a
formatter change, not for undoing this.

CSS Modules remain possible for anything awkward in utilities; Vite supports
both at once. This is not a door that closes.
