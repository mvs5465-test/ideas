# Overlord: Unified Codex Army Operating Model And Local Control Plane

## Summary

Unify the Codex army management model and the localhost control-plane idea into one proposal: `Overlord`, a lightweight local operator system that makes worker phases, status updates, and manager visibility part of the same product. The core recommendation is to treat the worker operating model and the dashboard as one design, not two separate ideas: workers move through explicit phases, report each transition to a localhost control plane, attach short phase notes and artifacts as they go, and give the operator a fast, intentionally good UI for steering the whole session without heavy infrastructure.

## Status

Proposed

## Problem

The current multi-worker pattern has two connected failures that should not be solved separately:

- the management model is too implicit, so workers duplicate discovery, collide on branches, and report status inconsistently
- the coordination surface is too weak, so operators must reconstruct state from chat, logs, git state, and PRs
- worker lifecycle changes are important, but they are not first-class events with validation, timing, or visibility
- blockers, ownership conflicts, and "what changed since the last check?" are expensive to answer quickly
- flat-file logging is good as a fallback, but weak as the live control path once several workers are active

PR #10 correctly pushes for a real army operating model. PR #11 correctly pushes for a localhost control plane. The stronger proposal is to combine them: the control plane should exist to enforce and visualize the operating model, and the operating model should be designed around what the control plane can validate and display.

## Proposal

Build toward `Overlord`, a localhost-first Codex operator control plane whose first job is to make the multi-worker operating model real.

The proposal has six core parts.

1. First-class worker phase model

Workers must move through an explicit state machine. Recommended base phases:

- `assigned`: manager created the task and ownership boundary
- `scouting`: worker is reading repo context and local instructions but not editing yet
- `planned`: worker has enough context to name its target artifact, risks, and next irreversible step
- `implementing`: worker is actively changing a scoped artifact
- `validating`: worker is running checks, reviewing diffs, or preparing the idea/PR handoff
- `blocked`: worker cannot continue because of a concrete dependency, conflict, or missing permission
- `handoff-ready`: worker is done locally and is ready for manager review or PR review
- `terminal`: worker has stopped because it completed, was superseded, was abandoned, or handed off

State transitions are not just labels. They are the operating model. The system should record when the transition happened, who reported it, what artifact the worker owns, and what the worker says comes next.

Transition policy should stay explicit so the extra phases add clarity instead of ceremony:

- `assigned -> scouting` is the normal entry path for every scoped task
- `scouting -> planned` is required when the worker still needs to lock the exact artifact, risk boundary, or next irreversible step before editing
- `scouting -> implementing` is allowed for simple, low-ambiguity tasks if the worker can already name the owned artifact and next irreversible step in the transition note
- `implementing -> validating` is required whenever the worker changed an artifact and needs a review pass, test pass, or handoff summary before marking ready
- `validating` may be skipped only if the worker stops in `blocked` or `terminal` before a normal handoff-ready path exists
- `blocked` is reachable from any active phase and should always name the minimal unblock needed

2. Control plane as the canonical live surface

Use one localhost service as the live coordination source of truth for active sessions:

- persist workers, assignments, heartbeats, phase transitions, blockers, notes, artifacts, repo ownership, and PR links in a local database
- expose a small localhost API so workers can register, post events, send heartbeats, and mark blockers or handoff readiness
- stream updates to the browser UI so the operator sees state changes immediately instead of polling several places
- keep Markdown export and `WORKER_LOG.md` style summaries as compatibility outputs, not the primary live control path

This live source of truth is operational, not constitutional. The control plane may own app-specific worker protocol docs, event schemas, and session data, but it must not replace the standing instruction hierarchy in `/Users/matthewschwartz/AGENTS.md`, `~/.codex/AGENTS.md`, or repo-local instructions. Those files remain the governance source of truth for how Codex should behave across repos.

3. Workers must report transitions to the dashboard

Every worker phase transition should generate one structured message that the control plane ingests over localhost HTTP. Workers should be explicitly expected to send these updates with `curl` or an equivalent simple HTTP client and treat a successful response as part of completing the phase transition.

The write path also needs an explicit localhost safety model so the stronger control surface stays constrained:

- bind only to `127.0.0.1` by default
- give each worker a scoped capability token so one worker cannot impersonate another by accident
- keep browser-side mutations behind operator session auth and CSRF protection, even on localhost
- validate repo paths against configured allowlists and reject oversized payloads
- preserve an append-only audit trail so worker updates and operator overrides stay attributable

Recommended MVP write schema:

```json
{
  "worker_id": "worker-20260315-example",
  "event_type": "phase_transition",
  "current_phase": "implementing",
  "previous_phase": "planned",
  "repo_path": "/Users/matthewschwartz/projects/ideas",
  "branch": "feat/example-slice",
  "worktree": "/Users/matthewschwartz/projects/_worktrees/example",
  "owned_artifact": "ideas/proposals/2026-03-15-overlord-codex-army-control-plane.md",
  "status_line": "rewriting rollout plan for parallel worker execution",
  "next_irreversible_step": "commit proposal updates after review pass",
  "blocker": null,
  "note": "working only in proposal body; no template changes",
  "timestamp": "2026-03-15T05:25:00Z"
}
```

Recommended MVP endpoints:

- `POST /api/workers/events` accepts the unified worker event message and returns success or a validation error
- `GET /api/workers` returns the current worker roster and latest known state for the operator
- `GET /api/workers/:worker_id` returns detailed worker state so the general can query one worker directly when needed

This turns "status updates" from optional chat behavior into the normal mechanism for coordination. The operator should be able to answer "who is active, stale, blocked, or ready?" from one board, and the general should have a clean read path for worker state without scraping logs.

4. Short bot notes in the control pane

Each phase should support one or more short worker-authored notes visible in the control pane, for example:

- scouting note: local instructions found, relevant files, and likely risks
- planned note: concrete implementation slice and conflict boundaries
- implementing note: artifact being edited and whether checks are pending
- blocked note: exact blocker and the minimal unblock needed
- validating note: checks run, failures, or open questions
- handoff-ready note: PR URL, summary of changes, and known follow-ups

These notes should stay short and phase-shaped, not become a second chat client. The point is a concise operator breadcrumb trail that matches the state machine.

5. Operator UI should be intentionally good and lightweight

The UI should not be an afterthought. It is the manager's working surface, so it should feel fast, dense, and pleasant to use without becoming a heavy frontend project.

Local precedent suggests the right split:

- `overlord` already points toward a lightweight localhost control plane with a small web UI and minimal dependency posture
- `snapreview` shows that a dense operator surface can still feel polished when the information hierarchy is deliberate
- deeper integration between the operating model and the app may be worth revisiting later, but that is out of scope for the MVP proposal

Recommendation:

- keep the product localhost-first and implementation-light
- prefer a simple web UI over a desktop shell
- bias toward server-rendered or otherwise lightweight app architecture
- keep the styling guidance stack-agnostic; choose the simplest approach that delivers a compact, readable operator board without forcing frontend build tooling unless the implementation proves it is worth the cost

The visual goal is an operator board that is compact, clear, and good enough to keep open all session.

6. Management protocol and app design should reinforce each other

The dashboard should directly support the army-management rules instead of acting as a passive status wall:

- show workers grouped by phase with stale-heartbeat highlighting
- make repo, branch, and worktree ownership conflicts obvious
- preserve scout-first planning for tasks that are still ambiguous
- distinguish active execution from blocked and waiting states
- make handoff-ready workers easy to review in a batch
- surface terminal outcomes clearly, including superseded or abandoned work

This is not "build a dashboard and hope behavior improves." It is "encode the management protocol so workers and operators can actually follow it."

## Scope

In scope:

- one unified idea for Codex army management plus localhost coordination UI
- explicit worker phases and transition rules
- structured worker updates and phase notes to the dashboard
- operator views for liveness, blockers, conflicts, and handoff readiness
- local-first persistence, event logging, and live updates
- UI guidance that is lightweight but intentionally polished
- Markdown export or session-summary compatibility with existing logs

Out of scope:

- implementing the app as part of this proposal document; implementation happens afterward if the proposal is accepted
- autonomous scheduling, worker spawning, or multi-host orchestration
- replacing git, GitHub, repo-local instructions, or human review
- building a heavyweight distributed system or message broker from day one

## Risks And Tradeoffs

- A stronger state machine adds process overhead for trivial one-worker tasks.
- If phase updates are too slow to send, workers will route around the system and the model will fail culturally before it fails technically.
- A polished UI can drift into operator-overreach if it encourages micromanagement instead of fast coordination.
- Localhost HTTP is pragmatic for workers and the browser, so prompt design and worker instructions need to strongly reinforce when to post updates, how to handle retries, and what counts as a valid successful response.
- Even on localhost, the API contract still needs worker identity scoping, CSRF discipline for browser writes, repo-path allowlists, and tight payload validation.
- If the UI implementation starts demanding a heavy frontend toolchain, the proposal should prefer a simpler rendering path until there is a concrete product need.

## Alternatives Considered

1. Keep the army operating model as process documentation only.

This helps managers prompt workers more consistently, but it leaves state reporting, liveness, and conflict visibility too manual.

2. Build the localhost dashboard without a strict operating model.

This creates a prettier status page, but not a reliable coordination system. Without defined phases and transition rules, the dashboard becomes an event bucket.

3. Keep flat files as the primary coordination path.

Useful as a fallback and export format, but too weak for validated transitions, live status, and conflict tracking.

4. Build a much heavier orchestration layer first.

Premature. The right first move is a local control plane that proves the worker model and the operator workflow on one machine.

Preferred approach: one integrated proposal where the worker lifecycle is the product model and the localhost control plane is the enforcement and visibility layer.

## Rollout Plan

1. Phase 1: unify the operating contract

- define the canonical worker phases, allowed transitions, and required transition payload
- standardize the worker output contract around repo, artifact, blocker, and next-step reporting
- package the worker operating contract as a Codex skill and, if useful, mirror app-specific protocol docs in the new `overlord` repo
- keep the durable Codex instruction hierarchy as the source of truth for cross-repo behavior; treat `overlord` docs as implementation-specific protocol documentation rather than governance replacement

2. Phase 2: build the control-plane backbone

- establish a local event/state schema for workers, heartbeats, blockers, notes, artifacts, and PR links
- add the localhost worker write path and live operator read path
- validate phase transitions instead of accepting arbitrary state text

3. Phase 3: ship the operator board

- add the main phase board, stale worker highlighting, conflict panels, and worker detail views
- show short phase notes directly in the control pane
- make handoff-ready and blocked queues visually distinct

4. Phase 4: make the workflow parallel-friendly

- shape task slices and worker instructions so separate workers can execute in parallel with minimal file overlap and clear ownership boundaries
- make conflicts obvious early by surfacing artifact ownership, overlapping repo scope, and blocked-on-worker dependencies in the board
- refine the UI using the lightweight `overlord` posture while preserving the option to stay server-rendered and dependency-light

## Success Criteria

- An operator can identify active, blocked, stale, validating, handoff-ready, and terminal workers from one localhost board in a few seconds.
- Workers consistently post phase transitions to the control plane instead of relying on ad hoc chat status.
- Each worker phase can carry a short, useful note in the control pane without turning the UI into a full chat system.
- Repo, branch, or worktree conflicts become visible shortly after they are reported rather than being discovered late in review.
- Multi-worker sessions with at least three workers feel easier to steer because the operating model and the dashboard are the same system.
- The resulting app direction stays lightweight and localhost-first instead of collapsing into an overbuilt orchestration platform.
