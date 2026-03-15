# Localhost Agent Control Plane For Codex Workers

## Summary

Replace ad hoc worker coordination through flat files with a local-first control plane app on `localhost` that gives Codex agents a structured write path and gives the operator a live status UI. The practical recommendation is a single local service with a SQLite state store, a localhost HTTP API for worker actions, and server-sent events for live UI updates. That keeps the system simple enough to run on one machine while materially improving coordination, safety, and observability over shared Markdown files.

## Status

Proposed

## Problem

The current file-based coordination model is cheap, but it breaks down once multiple workers are active:

- concurrent writes are awkward and easy to clobber
- state transitions are implicit instead of validated
- status polling is expensive because operators must inspect logs, files, branches, and PRs separately
- worker heartbeats, blockers, and artifact links are not naturally queryable
- there is no live UI for "who owns what right now?" or "which worker is stale?"

The existing Codex army operating-model proposal in this repo already identifies lifecycle states, output contracts, and shared logging as the management pain point. What is still missing is a purpose-built local system that makes those rules easy to follow.

## Proposal

Build a localhost-only agent control plane with one process owning coordination state and one browser UI for operators.

Recommended architecture:

1. Core model

- Use SQLite in WAL mode as the source of truth for workers, assignments, heartbeats, events, artifacts, blockers, and operator notes.
- Store both an append-only event table and a small set of materialized current-state tables.
- Keep the state model explicit: `assigned`, `scouting`, `implementing`, `blocked`, `handoff-ready`, `terminal`, plus timestamps, repo path, branch/worktree, owned artifact, and next step.

2. Local transport

- Default worker write path: HTTP on `127.0.0.1` with JSON endpoints such as `POST /workers/register`, `POST /events`, `POST /heartbeats`, `POST /blockers`, and `POST /artifacts`.
- Default UI push path: server-sent events from `GET /events/stream` so the browser gets live updates without polling.
- Optional later hardening: support a Unix domain socket listener in addition to TCP localhost, but do not make that the MVP dependency.

3. Operator UI

- Main board: workers grouped by lifecycle state with stale-heartbeat highlighting.
- Detail drawer: event timeline, current blocker, repo/branch/worktree, latest artifact, recent commands, and PR URL.
- Conflict views: duplicate repo ownership, overlapping branch claims, and idle/stuck workers.
- Controls: assign task, acknowledge blocker, mark terminal, and add operator notes.

4. Safety model

- Bind only to `127.0.0.1` by default.
- Give each worker a scoped capability token so one worker cannot impersonate another by accident.
- Keep browser mutations behind CSRF protection and operator session auth, even on localhost.
- Validate repo paths against configured allowlists and reject oversized payloads.
- Preserve an append-only audit trail so operator overrides and worker updates are attributable.

5. Observability

- Expose `/health` and `/metrics`.
- Emit structured JSON logs for worker actions and operator actions.
- Make tracing optional, not required, but keep spans around API handlers and DB writes if tracing is enabled.

Communication mechanism tradeoffs:

1. Flat file / append-only Markdown log

- Best for zero setup.
- Worst for concurrency, queryability, validation, and live UI.
- Keep only as an export or fallback, not the control path.

2. Direct SQLite writes from every worker

- Better atomicity than a flat file and excellent local queryability.
- But it couples every worker to schema details, makes auth weak, and increases lock/error handling burden inside workers.
- Not recommended as the primary integration surface.

3. Unix domain socket RPC

- Strong local-only boundary and avoids exposing a TCP port.
- Good long-term hardening option.
- Slightly worse ergonomics for browser access and ad hoc debugging than localhost HTTP.

4. Localhost HTTP API plus SSE

- Easiest path for workers (`curl`/small client), browser UI, and observability endpoints.
- Simpler operator debugging with ordinary web tooling.
- Needs explicit localhost binding and auth hygiene, but those controls are straightforward.
- Recommended for MVP.

5. Full message broker (`nats`, Redis streams, Kafka)

- Gives stronger async patterns and fan-out.
- Operationally too heavy for a single-machine local-first coordination app.
- Premature before one-process coordination proves insufficient.

Confirmed local precedent worth reusing:

- `ideas/proposals/2026-03-15-codex-army-management.md` already defines the lifecycle and output-contract language this app should encode.
- `cluster-query-router` shows that a small local HTTP service can successfully bundle HTML UI, `/health`, and `/metrics` in one app.
- `github-pr-slack-notifier` shows a good local observability baseline: JSON logs, Prometheus metrics, optional OTEL tracing, and fingerprinted state markers.
- `macdash-tui` shows the value of a dense, operator-first status surface rather than a settings-heavy admin tool.

Recommended MVP opinion:

- Start with one local service, one SQLite file, one browser UI, and one canonical event schema.
- Do not start with multi-host support, plugin systems, or autonomous scheduling.
- Make the app the source of live coordination truth, while allowing export back to Markdown for archival or sharing.

## Scope

In scope:

- local-only coordination service for Codex workers on one machine
- transport and state-model decisions
- operator status UI and conflict views
- worker registration, heartbeat, blocker, artifact, and terminal-state flows
- safety guardrails for localhost operation
- basic observability and auditability
- phased MVP slices that can be shipped incrementally

Out of scope:

- implementing the app in this proposal
- multi-host or internet-exposed orchestration
- replacing git, GitHub, or repo-local instructions
- full task planning automation or worker spawning
- heavyweight distributed systems infrastructure

## Risks And Tradeoffs

- A local service is more moving parts than a shared file, so the MVP must stay small or it becomes the new coordination burden.
- Localhost HTTP is ergonomically strong, but it is still an HTTP surface and needs real auth and CSRF discipline.
- SQLite is the right local-first default, but it is not ideal if the system later expands to many independent writer processes with long-running transactions.
- If the UI becomes too operator-driven, workers may stop reporting enough structured data and fall back to chat/manual habits.
- A strict state machine improves clarity, but may feel slower for tiny one-off tasks unless the update flow is fast.

## Alternatives Considered

1. Keep improving `WORKER_LOG.md` and other flat-file conventions.

This should still happen for low-friction compatibility, but it does not solve live queries, stale-heartbeat detection, or structured conflict tracking.

2. Build a desktop-native app first.

This could improve polish, but it increases implementation surface immediately. A localhost web app gets dense UI, browser debugging, and easy worker integration sooner.

3. Use direct SQLite as the worker API.

This is tempting for speed, but it pushes locking, schema drift, and identity concerns into every worker process.

4. Adopt a real broker from day one.

Useful only if the one-process model proves to be the bottleneck. For a localhost operator tool, that is an unjustified early cost.

Preferred approach: one localhost service with SQLite plus HTTP/SSE, then add Unix socket support or a broker only if real usage demonstrates the need.

## Rollout Plan

1. Slice 1: canonical event/state backbone

- define the worker, assignment, heartbeat, blocker, artifact, and terminal-event schema
- build SQLite tables and lifecycle validation rules
- add a minimal CLI or `curl`-friendly API contract for workers

2. Slice 2: operator board

- ship a single-page localhost UI showing active workers, stale workers, blockers, and latest artifacts
- add SSE-backed live updates
- add worker detail timeline and quick filters by repo, state, and staleness

3. Slice 3: conflict detection and operator actions

- detect overlapping repo/branch/worktree ownership
- allow operator reassignment, blocker acknowledgment, and terminal-state cleanup
- add audit entries for all operator overrides

4. Slice 4: hardening and ergonomics

- add scoped worker tokens, CSRF protection, payload limits, and path allowlists
- add export to Markdown for session summaries and compatibility with existing logs
- optionally add Unix socket mode if localhost TCP turns out to be an operational concern

## Success Criteria

- An operator can identify active, blocked, stale, and terminal workers from one localhost UI without manually scanning multiple files.
- Workers can report lifecycle changes, blockers, artifacts, and heartbeats through one structured local API instead of editing shared files.
- Repo/branch/worktree conflicts are visible within 5 seconds of being reported.
- The system exposes `/health`, `/metrics`, and structured logs from the first usable version.
- The MVP can run entirely on one machine with no external services beyond the local process and its SQLite database.
- Markdown/file-based coordination becomes an export or fallback path rather than the primary live coordination mechanism.
