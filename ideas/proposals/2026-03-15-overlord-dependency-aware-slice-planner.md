# Overlord Dependency-Aware Slice Planner

## Summary

Add a planning capability to Overlord that turns one manager mission into a small dependency graph of worker-sized slices with explicit ownership boundaries, validation gates, and merge order. This keeps parallel work useful instead of just visible.

## Status

Proposed

## Problem

The accepted Overlord control-plane proposal improves worker visibility, but it does not decide how a large task should be split before workers start. That leaves a repeat failure mode:

- managers assign overlapping work and discover conflicts late
- workers duplicate discovery because scope boundaries were implicit
- validation order is unclear when one slice depends on another
- "parallel" sessions collapse back into serial review because no one defined safe merge order up front

## Proposal

Add an Overlord slice-planning mode that produces a reviewable execution plan before delegation.

Minimum output:

- 3-8 concrete slices per mission
- one owner role per slice (`captain` or `soldier`)
- exact artifact boundary per slice (repo, files, or PR target)
- dependency edges between slices
- validation gate for each slice
- handoff order for review and merge

MVP behavior:

1. Manager submits a mission plus repo scope.
2. Overlord proposes a slice graph with `blocked_by` edges.
3. Overlord rejects plans that leave multiple slices owning the same file set unless the manager explicitly overrides it.
4. Overlord marks slices as ready only when upstream dependencies are complete and required evidence is attached.
5. Overlord shows the graph in the operator UI and exports a compact worker prompt per slice.

## Scope

In scope:
- mission decomposition into concrete slices
- dependency and ownership modeling
- per-slice validation requirements
- review/merge order generation
- manager override for intentional overlap

Out of scope:
- autonomous code changes
- automatic worker spawning
- replacing human review
- broad portfolio planning across repos

## Risks And Tradeoffs

- Bad slicing logic can add process instead of removing it.
- File-level ownership checks are imperfect when work overlaps by behavior rather than path.
- Managers may trust the plan too much unless the UI makes assumptions visible.
- Some tasks are genuinely single-threaded; forcing 3+ slices would be noise.

## Alternatives Considered

1. Keep planning manual in chat.
- Lowest implementation cost, but overlap and dependency mistakes remain common.

2. Let workers negotiate boundaries after they start.
- Flexible, but expensive because conflict is discovered after context loading and edits.

3. Add full autonomous scheduling first.
- Premature before Overlord can reliably produce reviewable slice graphs.

Preferred approach: a manager-reviewed slice planner that improves parallel execution without taking control away from the operator.

## Rollout Plan

1. Start with one repo, one mission, and manual manager approval of every generated slice graph.
2. Add file-scope overlap checks and required validation templates per slice.
3. Surface ready/blocked slice states in the existing Overlord board.
4. Measure whether multi-worker sessions finish with fewer conflicts and less serial cleanup.

## Success Criteria

- In sessions with 3 or more workers, late-discovered file ownership conflicts drop by at least 50%.
- At least 80% of slices can be assigned without manager rewording after the first plan draft.
- Each slice reaches review with a named validation gate and artifact boundary.
- Managers can identify the next unblocked slice and required merge order from one screen.
