# Codex-Led Multi-Model Code Review

Use this when the user asks for `consensus-code-review`, `code-review`, or a multi-model code review from a Codex session.

Follow the same configuration, session, collection, comparison, synthesis, convergence, and final-output rules as `../../consensus-plan-review/references/command.md`, with the changes below.

## Session

- Session directory: `/tmp/codex-consensus-code-review-XXXXXX`
- Progress file: `data/scratch/active-progress-codex-consensus-code-review-{SESSION_ID}.md`
- Codex writes its own review to `$SESSION_DIR/codex.md`
- Final draft path: `$SESSION_DIR/draft.md`

## Input

`$ARGUMENTS` is the review target: staged changes, unstaged changes, last commit, last N commits, a branch diff, file paths, or a PR number.

If empty:

1. Review `git diff HEAD` plus `git diff --cached`.
2. If empty, review `git diff main...HEAD`.
3. If still empty, ask the user what code to review.

Resolve common targets:

- `staged changes`: `git diff --cached`
- `unstaged changes`: `git diff`
- `working tree`: `git diff HEAD`
- `last commit`: `git show --stat --patch HEAD`
- `last N commits`: `git log --oneline -n N` and `git diff HEAD~N...HEAD`
- branch diff: `git diff main...HEAD` unless the user names another base
- file paths: inspect those files plus relevant callers and tests

Support `--dirs /path/a,/path/b` exactly like plan review.

## CodeRabbit

If `command -v coderabbit` succeeds, run CodeRabbit as a supplementary static-analysis reviewer:

```bash
coderabbit review --plain > "$SESSION_DIR/coderabbit.md" 2>&1
```

Include CodeRabbit findings in comparison and synthesis, but do not count CodeRabbit toward quorum and do not include it in convergence rounds.

## Shared Prompt

Write `$SESSION_DIR/prompt.md`:

```markdown
I need a thorough code review. The review target is:

---
{DESCRIPTION}
---

To see the changes, run:

`{GIT COMMAND OR FILE LIST}`

You have full access to the codebase at `{REPO_DIR}`. Explore it directly. Read changed files, related callers, tests, imports, public APIs, configs, schemas, and existing patterns before reviewing. Do not review the diff in isolation.

{EXTRA DIRECTORIES, IF ANY}

For each real issue found, provide:
1. Severity: CRITICAL / MAJOR / MINOR / NIT
2. Category: Bug, Security, Performance, Logic Error, Code Quality, Design, Testing, Documentation
3. Location: file path and line number
4. Description: what is wrong and why it matters
5. Suggestion: how to fix it

Also include:
- Overall assessment
- Missing tests
- Unintended consequences: broken callers, stale tests, downstream consumers, contract changes, migrations, performance, security, and operational concerns

If no unintended consequences are found, state exactly: "None found - searched callers, tests, and downstream consumers."

Flag real issues only. Prioritize correctness, safety, behavioral regressions, and missing tests.
```

Do not mention model providers or model names in prompts sent to external tools.

## Codex Review

Codex independently reviews the target and writes `$SESSION_DIR/codex.md` while external reviewers run.

Use the standard code-review stance:

- findings first
- ordered by severity
- grounded in file and line references
- summarize only after issues
- if no issues are found, say so and mention test gaps or residual risk

## Compare And Synthesize

Read `codex.md`, each external model file, and `coderabbit.md` if present.

Show:

- consensus issues
- valuable unique findings
- disagreements or false positives
- missing tests
- unintended consequences
- comparison table with Codex, CodeRabbit if present, and participating models

The synthesized review should include:

- top findings with severity
- positive observations only if multiple reviewers noted them
- missing tests
- unintended consequences
- final recommendation
- finding attribution table

## Convergence

Use this convergence prompt:

```markdown
Here is a synthesized code review based on multiple independent reviews:

---
{DRAFT REVIEW}
---

Review this synthesis. Start your response with one of:
- APPROVE - if the review accurately captures the real issues
- CHANGES NEEDED - if something is wrong, missing, or overstated

Only raise issues that materially affect correctness, severity, or usefulness.
```

Run convergence for external model reviewers only. CodeRabbit does not participate in convergence.
