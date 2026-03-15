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

2. Control plane as the canonical live surface

Use one localhost service as the live coordination source of truth for active sessions:

- persist workers, assignments, heartbeats, phase transitions, blockers, notes, artifacts, repo ownership, and PR links in a local database
- expose a small localhost API so workers can register, post events, send heartbeats, and mark blockers or handoff readiness
- stream updates to the browser UI so the operator sees state changes immediately instead of polling several places
- keep Markdown export and `WORKER_LOG.md` style summaries as compatibility outputs, not the primary live control path

3. Workers must report transitions to the dashboard

Every worker phase transition should generate a structured dashboard update. Minimum transition payload:

- worker ID
- current phase
- previous phase
- repo path
- branch or worktree
- owned artifact
- short status line
- next irreversible step
- timestamp

This turns "status updates" from optional chat behavior into the normal mechanism for coordination. The operator should be able to answer "who is active, stale, blocked, or ready?" from one board.

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
- `snapreview` shows the strongest recent precedent for polished operator-facing styling with Tailwind CSS plus daisyUI

Recommendation:

- keep the product localhost-first and implementation-light
- prefer a simple web UI over a desktop shell
- bias toward server-rendered or otherwise lightweight app architecture
- if richer styling is needed, reuse the proven Tailwind CSS plus daisyUI approach from local precedent rather than inventing a bespoke design system

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

- implementing the app in this proposal
- autonomous scheduling, worker spawning, or multi-host orchestration
- replacing git, GitHub, repo-local instructions, or human review
- building a heavyweight distributed system or message broker from day one

## Risks And Tradeoffs

- A stronger state machine adds process overhead for trivial one-worker tasks.
- If phase updates are too slow to send, workers will route around the system and the model will fail culturally before it fails technically.
- A polished UI can drift into operator-overreach if it encourages micromanagement instead of fast coordination.
- Localhost HTTP is pragmatic for workers and the browser, but it still needs basic auth, CSRF, and payload discipline.
- Tailwind CSS plus daisyUI is a good styling precedent for operator surfaces, but the repo should still avoid turning a small control plane into a frontend-heavy build chain unless the value is clear.

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
- keep `WORKER_LOG.md` compatible as a summary/export surface

2. Phase 2: build the control-plane backbone

- establish a local event/state schema for workers, heartbeats, blockers, notes, artifacts, and PR links
- add the localhost worker write path and live operator read path
- validate phase transitions instead of accepting arbitrary state text

3. Phase 3: ship the operator board

- add the main phase board, stale worker highlighting, conflict panels, and worker detail views
- show short phase notes directly in the control pane
- make handoff-ready and blocked queues visually distinct

4. Phase 4: tighten ergonomics and safety

- add worker identity tokens, CSRF/session protection, and path allowlists
- add compact filters, keyboard-friendly operator actions, and Markdown export
- refine the UI using the lightweight `overlord` posture and the Tailwind CSS plus daisyUI precedent proven in `snapreview`

## Success Criteria

- An operator can identify active, blocked, stale, validating, handoff-ready, and terminal workers from one localhost board in a few seconds.
- Workers consistently post phase transitions to the control plane instead of relying on ad hoc chat status.
- Each worker phase can carry a short, useful note in the control pane without turning the UI into a full chat system.
- Repo, branch, or worktree conflicts become visible shortly after they are reported rather than being discovered late in review.
- Multi-worker sessions with at least three workers feel easier to steer because the operating model and the dashboard are the same system.
- The resulting app direction stays lightweight and localhost-first instead of collapsing into an overbuilt orchestration platform.
