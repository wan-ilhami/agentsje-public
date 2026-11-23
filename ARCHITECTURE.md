# Architecture

```
 browser (Next.js + Three.js)          backend (Bun)                         on this machine
 ┌──────────────────────────┐ HTTP/SSE ┌────────────────────────────┐
 │ office: 3D floor, desks  │◄────────►│ http      routes, auth     │
 │ panels: tasks, diffs,    │          │ app       task engine      │──► Postgres (task tree)
 │ templates, usage         │          │ domain    pure rules       │──► coding-agent CLI (one process per run)
 └──────────────────────────┘          │ infra     adapters         │──► git worktrees per task
                                       └────────────────────────────┘
 data on disk: agents/<dept>/<id>.md · skills/<name>/SKILL.md · rules/ · commands/
```

## Backend — ports and adapters

- **domain** — pure logic: loading agents, building prompts, parsing delegation and file hand-over lines, schedules,
  usage and budget maths, auth token checks. No IO.
- **app** — the task engine. Queues work per agent (one task at a time), runs delegations in parallel, and resumes
  the parent to synthesise once every child has settled. Talks to the outside only through interfaces.
- **infra** — the only code that touches Postgres, the agent CLI, git worktrees, the filesystem and server-sent events.
- **http** — routing, bearer-token auth with admin and viewer roles, rate limiting, request logging.

## Task lifecycle

`queued → working → (waiting → synth → working →) done | failed`

A gated coding task first runs in plan mode and waits in `approval`; approving it reruns the same agent in code mode
with the plan attached. Every run has a timeout and can be cancelled; cancelling a parent cancels its open children.

## Safety rails

- Agents run sandboxed in plan mode by default; write access is granted per agent, never globally.
- Task repositories must sit inside an allow-listed set of roots.
- Delegation depth, fan-out and chain length are capped, and repeated steps are refused.
- Webhooks go to https (or localhost) only; private address ranges are refused and the resolved IP is pinned.
- Without an API token the server binds to localhost only.

## Frontend

A Next.js client with a Three.js office scene. State comes from the REST API plus a server-sent event stream that
resyncs after reconnects. All logic that can be pure (formatting, filtering, diff parsing, pathfinding) lives in plain
modules with unit tests; end-to-end specs drive the office in a real browser.
