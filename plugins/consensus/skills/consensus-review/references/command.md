# Codex-Led Multi-Model General Review

Use this when the user asks for `consensus-review`, a multi-model document review, a spec review, or a general technical consensus review from a Codex session.

Follow the same configuration, session, collection, comparison, synthesis, convergence, and final-output rules as `../../consensus-plan-review/references/command.md`, with the changes below.

## Session

- Session directory: `/tmp/codex-consensus-review-XXXXXX`
- Progress file: `data/scratch/active-progress-codex-consensus-review-{SESSION_ID}.md`
- Codex writes its own review to `$SESSION_DIR/codex.md`
- Final draft path: `$SESSION_DIR/draft.md`

## Input Detection

`$ARGUMENTS` can be:

- a document path
- `the plan` or `latest plan`, resolved to the newest `~/.claude/plans/*.md`
- a free-text question, proposal, architecture choice, or spec

If the input is empty:

1. Use the current conversation document or proposal if one exists.
2. Otherwise check the newest `~/.claude/plans/*.md`.
3. If neither exists, ask the user what to review.

If the document is outside the repo and external CLIs may not be able to access it, copy it to `$SESSION_DIR/input.md` and refer to that path in prompts.

Support `--dirs /path/a,/path/b` exactly like plan review.

## Shared Prompt: Document

For a document or plan, write `$SESSION_DIR/prompt.md`:

```markdown
I need a thorough review of this document or plan:

`{FILE_PATH}`

Read it completely. Verify claims against the actual codebase where relevant. Challenge assumptions.

{EXTRA DIRECTORIES, IF ANY}

Evaluate:
1. Accuracy
2. Completeness
3. Feasibility
4. Internal consistency
5. Actionability
6. Risks and assumptions
7. Structure
8. Unintended consequences

Return specific issues with severity, location, evidence, and suggested fix.
```

## Shared Prompt: General Question

For a general question or proposal, write `$SESSION_DIR/prompt.md`:

```markdown
I need your assessment of the following:

---
{USER DESCRIPTION, VERBATIM}
---

You have full access to the codebase. Explore relevant files if the question depends on implementation details.

{EXTRA DIRECTORIES, IF ANY}

Evaluate correctness, completeness, quality, risks, unintended consequences, and alternatives. Be specific and opinionated.
```

Do not mention model providers or model names in prompts sent to external tools.

## Codex Review

Codex independently reviews the document, plan, or question and writes `$SESSION_DIR/codex.md` while external reviewers run.

## Compare And Synthesize

Show:

- consensus issues
- valuable unique findings
- conflicts
- assumptions that need user confirmation
- unintended consequences
- comparison table with Codex and each participating model

The synthesized review should include:

- final assessment
- issues by severity
- recommended changes
- unresolved disagreements
- attribution table

## Convergence

Use this convergence prompt:

```markdown
Here is a synthesized review based on multiple independent reviews:

---
{DRAFT REVIEW}
---

Review this synthesis. Start your response with one of:
- APPROVE - if the review is accurate and useful
- CHANGES NEEDED - if something is wrong, missing, or overstated

Only raise issues that materially affect correctness, feasibility, or usefulness.
```
