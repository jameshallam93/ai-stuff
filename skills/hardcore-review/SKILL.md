---
name: hardcore-review
description: Run a multi-perspective PR review by understanding the current change, selecting only the relevant review personas, delegating to them with subagents, filtering their findings, and producing a prioritized markdown summary.
metadata:
  short-description: Orchestrated multi-skill PR review
---

# Hardcore Review

Use this skill when the user wants a serious review of the current pull request rather than a single-perspective pass. This skill is an orchestrator. It reads the PR, understands the context, selects the right review skills for the change, dispatches them in parallel, evaluates the findings, and returns only the useful results.

## What This Skill Does

This skill:

- Understands the current PR and its context
- Selects the relevant review skills from the local review skill set
- Spawns one subagent per selected review skill
- Collects and filters findings from those subagents
- Presents a consolidated review to the user
- Writes a prioritized markdown summary file

The point is selectivity, not brute force. Do not run every review persona on every PR.

## Available Review Skills

Check the local `reviews/` directory and treat each skill there as a candidate reviewer. At the time of writing, this includes:

- `super-senior`
- `intrepid-junior`
- `savvy-product-manager`
- `pedantic-nitpicker`
- `paranoid-sre`
- `test-maximalist`

If new review skills are added later, include them in the candidate set and apply the same selection discipline.

## Intake Workflow

Before choosing reviewers:

1. Read the current PR diff
2. Read the PR title, description, and comments if available
3. Identify the main change types
4. Identify the intended behaviour, user impact, and operational impact
5. Identify what parts of the codebase are touched

Use the best available PR source:

- Prefer connected PR metadata and diff tools when available
- Fall back to local git diff and branch context when PR metadata is unavailable

Do not start subagent reviews until the change is understood well enough to route accurately.

## Selection Rules

Choose review skills based on the actual content of the PR.

### `super-senior`

Use for:

- Most substantive code changes
- Architectural or non-trivial implementation changes
- Changes with meaningful tradeoffs

Skip for:

- Pure docs changes
- Tiny mechanical changes with no real design or implementation judgment

### `intrepid-junior`

Use for:

- Changes that introduce new concepts, abstractions, or workflows
- Code likely to be read or modified by a wide set of engineers
- Refactors where readability and onboarding matter

Skip for:

- Purely internal infrastructure changes with little onboarding relevance
- Tiny localized fixes where readability is not really in question

### `savvy-product-manager`

Use for:

- User-facing UI changes
- Documentation changes
- Copy changes
- API or behaviour changes that affect user workflows or support expectations

Skip for:

- Pure backend refactors with no product-facing consequences

### `pedantic-nitpicker`

Use for:

- Changes where polish, wording, formatting, or local consistency materially matter
- Docs, copy, templates, or UI-heavy PRs
- Final-pass review when a nit-only sweep is likely to add signal

Skip for:

- Large risky changes where nit review would mostly add noise
- Any PR where no worthwhile nit surface is likely

### `paranoid-sre`

Use for:

- Runtime behaviour changes
- Infra, deployment, queue, job, persistence, network, or dependency changes
- Changes that affect observability, retries, rollout, or failure handling

Skip for:

- Pure docs or copy changes
- Isolated presentation changes with no operational effect

### `test-maximalist`

Use for:

- Code changes with meaningful behaviour
- Bug fixes
- Refactors that may weaken regression protection
- Any PR where the tests are a major part of confidence

Skip for:

- Pure docs or copy changes
- Very small non-behavioural changes where no test strategy judgment is needed

## Minimum Reviewer Set

Do not use fewer reviewers than the change justifies, but do not over-review.

Default guidance:

- Substantive code PR: usually `super-senior` plus `test-maximalist`
- User-facing PR: usually `savvy-product-manager`, often plus `super-senior` or `intrepid-junior`
- Operational PR: usually `paranoid-sre`, often plus `super-senior` and `test-maximalist`
- Docs-heavy PR: usually `savvy-product-manager`, optionally `pedantic-boob`

If a skill would add mostly boilerplate findings, leave it out.

## Subagent Workflow

Spawn one subagent per selected review skill.

Each subagent should receive:

- The PR title and description if available
- The relevant diff or changed files
- Any important surrounding context discovered during intake
- Clear instruction to review through exactly one persona only

Each subagent should be told to:

- Produce findings first
- Stay strictly within the scope of its assigned skill
- Reference files and lines where possible
- Avoid repeating generic praise or summary filler
- Return “no substantive findings” when appropriate

Prefer parallel execution when tooling allows it.

## Aggregation Rules

When subagents report back:

- Deduplicate overlapping findings
- Drop findings that are incorrect, weak, out of scope, or not grounded in the actual diff
- Resolve conflicts by favouring the finding that best matches the PR context and the selected skill scope
- Preserve low-severity nit findings only when they are still worth the user’s attention after higher-signal findings are considered

The main agent is the editor, not a courier. Do not blindly forward subagent output.

## Prioritization Rules

Present the final findings in priority order:

1. High-confidence correctness, safety, or user-impact issues
2. Material maintainability, operability, or test-confidence issues
3. Readability and onboarding issues worth addressing
4. Nits that remain worth mentioning

If a chosen reviewer returns nothing useful, omit it from the final presentation.

## Output To User

Return:

- A short note on which review skills were used and why
- The consolidated findings, ordered by priority
- Any important open questions or assumptions

Do not organize the final answer by reviewer unless the user explicitly asks for that. Organize by issue priority instead.

## Markdown Artifact

After consolidating the review, create a markdown file in the repo root named:

`hardcore-review-<branch-or-pr-identifier>.md`

The file should contain:

- PR identifier or branch name
- Review skills used
- Short context summary
- Prioritized findings
- Open questions or assumptions
- A note when a candidate reviewer was intentionally skipped because it was not relevant

If no stable PR identifier is available, use the current branch name. If that is also awkward, use `hardcore-review-summary.md`.

## Failure Handling

If the PR cannot be fully determined:

- State what context is missing
- Review from the best available local diff
- Be explicit about the confidence limit this creates

If subagents are unavailable:

- Run the same selection logic locally
- Perform the selected reviews sequentially
- Still produce the same consolidated output and markdown artifact

## Style Guidance

- Be selective
- Be evidence-driven
- Prefer signal over coverage theatre
- Keep subagents tightly scoped
- Keep the final output shorter than the combined raw findings
- Treat the markdown artifact as a clean deliverable, not a dump of notes

## Example Prompts

- Run a hardcore review on the current PR.
- Use hardcore-review to decide which reviewers matter for this diff.
- Review this PR with the relevant personas only, then write a prioritized markdown summary.
