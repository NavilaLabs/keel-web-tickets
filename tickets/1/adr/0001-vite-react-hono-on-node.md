# 0001. keel-web uses Vite + React + Hono on Node

Status: accepted
Date: 2026-09-19
Ticket: 1
Block: b1

## Context
keel-web is a web UI for keel. v1 needs a browser chat with Claude Code (streaming), view-only ticket and pull request views (GitHub, Jira, Bitbucket), LikeC4 diagram rendering, and a per-ticket phase indicator. The UI look is undecided. The developer wants it simple and leaned towards Deno + Fresh, but named smooth operation as more important than the runtime.

## Options
### A. Deno + Fresh 2.x
Simplest to start and closest to the initial preference. Fresh islands use Preact, LikeC4's diagram library is React-based, and Preact compatibility is undocumented and unverified. The Claude Agent SDK under Deno is also undocumented. Node would probably still be needed in the container for the LikeC4 CLI and Vite plugin, giving two runtimes.
### A'. Deno runtime with Vite + React + Hono
Keeps Deno and avoids Preact. The Agent SDK under Deno stays unverified. Not researched beyond that.
### B. Vite + React + Hono on Node
LikeC4's official Vite plugin and React components work as documented, and the Agent SDK runs on its primary runtime. No single official combined template, and routing and the client/server API contract are written by hand. Two packages or a monorepo.
### C. SvelteKit (adapter-node)
Mature. React inside Svelte needs a wrapper or a separate bundle for LikeC4, and long-lived connection behaviour on adapter-node was not verified.
### D. Next.js (App Router)
LikeC4 works, but it is the heaviest option, Vercel-oriented, and conflicts with "keep it simple".

## Decision
Option B: Vite + React for the client and Hono on Node for the server. It has the fewest unverified assumptions for LikeC4 and the Agent SDK, and it forces the least on the UI look. The Deno preference was explicitly traded for smoother operation.

## Consequences
One runtime (Node) in the devcontainer. LikeC4 and the Agent SDK are used on their documented paths. The starter template is assembled from the Vite and Hono templates rather than one official template. Deno and Fresh are foreclosed unless a later ADR supersedes this one.
