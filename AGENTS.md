# AGENTS.md

Instructions for human + AI contributors in this repository.

## Purpose

This repo is for collaborating on ideas via pull requests. Every idea should be reviewable, challengeable, and mergeable as a concrete artifact.

## Contribution Rules

1. Propose ideas as Markdown files under `ideas/proposals/`.
2. Use `ideas/templates/feature-idea.md` as the starting structure for feature ideas.
3. Keep writing concise and concrete. Prefer tradeoffs, risks, and measurable success criteria over abstract brainstorming.
4. Use feature branches and pull requests. Do not push directly to `main`.
5. PR titles and commit messages must use Conventional Commits.

## Required PR Description Preface For Feature Ideas

For PRs that propose feature ideas, include this exact line near the top of the PR description:

`Reviewer special instructions: play devil's advocate. Don't just approve because it sounds good. Try to poke meaningful holes in the plan. Suggest improvements or ideas that may not be present.`

Keep it visible and unedited in the PR description unless the repository owner asks otherwise.

## Reviewer Expectations

1. Pressure-test assumptions.
2. Call out unclear scope, hidden complexity, and missing rollout/risk details.
3. Request changes when an idea is not yet actionable.
4. Approve only when the proposal has clear value, constraints, risks, alternatives, and success criteria.

## Author Expectations

1. Treat requested changes as part of refinement, not rejection.
2. Update the proposal in follow-up commits instead of force-pushing rewritten history.
3. Resolve each review thread with concrete changes or explicit rationale.
