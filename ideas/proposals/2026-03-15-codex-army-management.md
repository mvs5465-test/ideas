# Codex Army Management Operating Model

## Summary

Standardize how a manager runs multiple delegated Codex workers so parallel execution stays useful instead of collapsing into branch conflicts, unclear ownership, and slow status gathering. The proposal defines a lightweight operating model around worker lifecycle states, scout-to-implementation waves, disciplined status logging, and explicit conflict-handling rules so multi-worker sessions scale beyond ad hoc prompting.

## Status

Proposed

## Problem

Delegating to several Codex workers can create real throughput, but the management overhead rises fast:

- workers overlap on the same repo or branch and block each other
- branch and worktree conflicts are discovered late instead of at assignment time
- managers spend too much time polling for status because progress reporting is inconsistent
- early research and later implementation get mixed together, so multiple workers duplicate discovery work
- useful session lessons stay in chat history instead of becoming repeatable management rules

The result is a brittle "ideas army" pattern where parallelism exists, but manager attention becomes the bottleneck.

## Proposal

Adopt a concrete Codex army operating model with five parts.

1. Worker lifecycle contract

Define explicit worker states and required transitions:

- `assigned`: manager issued task, repo, worker ID, and success criteria
- `scouting`: worker gathers repo context and local instructions, but does not edit yet
- `implementing`: worker owns a scoped artifact or code path and is expected to edit
- `blocked`: worker cannot proceed because of a specific dependency, conflict, or missing permission
- `handoff-ready`: worker finished local changes and is ready for review or PR
- `terminal`: worker stopped after PR creation, rejection, or explicit abandonment

Each worker must report the current state, the concrete artifact it is touching, and the next irreversible step before taking it.

2. Scout -> implementation waves

Split larger efforts into two management waves instead of sending every worker straight to editing:

- Wave 1 scouts map repos, constraints, risks, and likely edit surfaces
- Manager consolidates findings into a short execution plan
- Wave 2 implementers get narrower instructions with scoped ownership and conflict boundaries

This keeps exploration parallel while reducing duplicate edits and avoids multiple workers discovering the same repo-local rules independently.

3. Status and logging discipline

Require all workers to write compact terminal summaries to a shared manager-visible log such as `WORKER_LOG.md` and use a stable output contract:

- status
- resolved repo path
- files changed
- summary of work or finding
- commands run
- blockers
- PR URL or artifact path

Manager status checks should use the fastest narrow signal first:

- log heartbeat
- branch existence
- one artifact-shaped verification

Do not normalize slow foreground polling as the default management loop.

4. Branch and worktree conflict handling

Add assignment-time rules that force conflict checks before implementation starts:

- one worker owns one repo branch unless the manager explicitly declares a shared branch model
- when multiple workers must touch the same repo, assign separate worktrees or separate files with a named owner
- workers must surface current branch, dirty state, and upstream divergence before creating edits
- if a worker discovers unexpected user changes in its target files, it stops and reports the conflict instead of improvising around it

Managers should prefer repo-level sharding first, worktree-level sharding second, and same-branch multi-writer setups only as a last resort.

5. Manager ergonomics

Give the manager a small standard toolkit instead of relying on memory:

- manager-scoped worker IDs
- a reusable delegated-worker prompt skeleton
- a compact checklist for repo resolution, instruction loading, branch setup, PR requirements, and terminal logging
- a default response format for "how are workers doing?" that favors `alive/dead/unclear` plus one concrete artifact

The goal is not heavy process. It is to make parallel Codex sessions easier to steer under time pressure.

## Scope

In scope:

- operational rules for delegated Codex workers
- worker-state model and output contracts
- scout and implementation wave planning
- branch/worktree ownership rules
- manager-facing ergonomics and status habits
- lightweight shared logging expectations

Out of scope:

- building a full orchestration service or scheduler
- automatic worker spawning infrastructure
- replacing GitHub, git worktrees, or repo-local contributor instructions
- measuring token cost or model performance as the primary goal

## Risks And Tradeoffs

- More structure can slow down very small tasks where one worker would have been enough.
- Logging discipline only works if workers actually follow the contract.
- Scout waves add an extra management step before code lands.
- Strict branch ownership reduces accidental conflicts but can underuse available workers if tasks are oversharded.
- The model improves manager clarity, but not every repo will justify the same amount of ceremony.

## Alternatives Considered

1. Keep current ad hoc delegation.

This has the lowest upfront process cost, but it does not scale well once multiple workers share repos or the manager needs fast status signals.

2. Use one worker per repo and avoid same-repo parallelism entirely.

This is simpler, but leaves throughput on the table for larger repos with naturally separable workstreams.

3. Build a heavy orchestration layer first.

This could eventually help, but it is premature before the team agrees on the operating model and failure cases worth automating.

Preferred approach: standardize the management protocol first, then automate the parts that prove repeatedly useful.

## Rollout Plan

1. Phase 1: Manager template

- write a reusable delegated-worker prompt template with required output contract and terminal log instructions
- define the canonical worker lifecycle states
- adopt manager-scoped worker IDs for all delegated sessions

2. Phase 2: Scout and ownership rules

- require scout waves for any effort involving multiple repos or likely same-repo collisions
- add assignment-time checks for branch, dirty-tree, and worktree ownership
- document when to shard by repo, branch, or worktree

3. Phase 3: Status discipline

- standardize `WORKER_LOG.md` append format
- define the manager fast-path status check order
- require workers to report blockers as soon as they hit them instead of waiting for a full summary

4. Phase 4: Ergonomic tooling

- add small helper scripts or templates only after the operating rules are stable
- consider lightweight automation for worker registration, heartbeat collection, and artifact lookup

## Success Criteria

- Multi-worker sessions can be run with at least three delegated workers without unplanned branch collisions.
- Manager status checks usually complete from shared log plus one narrow verification instead of broad repo scanning.
- Workers consistently return terminal summaries with repo path, artifacts, blockers, and PR URLs when applicable.
- Same-repo tasks are more often split into scout and implementation waves instead of duplicate exploratory edits.
- Manager prompts become shorter over time because the lifecycle, logging, and conflict rules are already standardized.
