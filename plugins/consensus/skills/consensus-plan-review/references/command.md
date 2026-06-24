# Codex-Led Multi-Model Plan Review

Use this when the user asks for `consensus-plan-review`, `claude-consensus`, `plan-review`, a multi-model plan review, or a consensus review of an implementation plan from a Codex session.

This is the Codex version of the Claude Consensus plan-review workflow. Codex is the lead reviewer and synthesizer, so never invoke Codex/GPT as an external model.

## Configuration

1. Read `~/.codex/consensus.json`.
2. If missing, read `plugins/consensus/consensus.codex.config.json` from the plugin root.
3. If neither exists, stop and ask the user to run `consensus-setup`.
4. Parse JSON and filter `models[]` to `enabled: true`.
5. Drop any model whose `id` is in `lead.excluded_model_ids`, whose id is `gpt` or `codex`, or whose command starts with `codex`.
6. Validate each remaining model has non-empty `id`, `name`, and `command`.
7. Use `min_quorum` exactly as configured. If it is missing or invalid, default to strict majority of Codex plus enabled external models.

Source only the OpenRouter key when needed:

```bash
if [ -f ~/.codex/.env ] && grep -q '^OPENROUTER_API_KEY=.\+' ~/.codex/.env; then
  export OPENROUTER_API_KEY="$(grep '^OPENROUTER_API_KEY=' ~/.codex/.env | cut -d= -f2- | tr -d '"')"
elif [ -f ~/.claude/.env ] && grep -q '^OPENROUTER_API_KEY=.\+' ~/.claude/.env; then
  export OPENROUTER_API_KEY="$(grep '^OPENROUTER_API_KEY=' ~/.claude/.env | cut -d= -f2- | tr -d '"')"
fi
```

Preflight:

- commands starting with `kilo` require `command -v kilo` and a non-empty `OPENROUTER_API_KEY`
- commands starting with `qwen` require `command -v qwen`
- commands starting with `codex` must be skipped
- never use `--yolo`

Count available external models plus Codex. If available participants are fewer than `min_quorum`, abort with a clear summary of unavailable models and ask the user to reconfigure.

## Input

`$ARGUMENTS` is the task or plan to review.

If it is empty:

1. Use the current conversation plan if one exists.
2. Otherwise check the newest `~/.claude/plans/*.md`.
3. If neither exists, ask the user for the task or plan.

Support `--dirs /path/a,/path/b`:

1. remove the flag from the user task
2. validate each directory exists
3. include valid extra directories in the shared prompt
4. pass directory flags only to CLIs that support them

Directory flags:

- native Qwen: `--include-directories /path/a,/path/b`
- Kilo/OpenRouter: no CLI flag; include paths in the prompt

## Session

Create a session directory:

```bash
SESSION_DIR=$(mktemp -d /tmp/codex-consensus-plan-review-XXXXXX)
SESSION_ID=$(basename "$SESSION_DIR" | sed 's/^codex-consensus-plan-review-//')
mkdir -p data/scratch
```

Write progress to:

```text
data/scratch/active-progress-codex-consensus-plan-review-{SESSION_ID}.md
```

The progress file should include command, status, session directory, target, model list, quorum, and phase checklist. Mark it `IN_PROGRESS`, then `COMPLETED` or `FAILED`.

## Shared Prompt

Write `$SESSION_DIR/prompt.md`:

```markdown
I need to accomplish the following task:

---
{USER TASK OR PLAN, VERBATIM}
---

You have full access to the codebase at `{REPO_DIR}`. Explore it directly. Read relevant files, understand existing patterns and architecture, and base your plan on the actual implementation.

{EXTRA DIRECTORIES, IF ANY}

Create a detailed implementation plan. Include:
1. Files to modify/create with specific paths
2. Step-by-step approach with code-level details
3. Key architectural decisions and why
4. Edge cases and failure modes
5. Verification steps

Also check for unintended consequences: broken callers, stale tests, downstream consumers, API/CLI/config contract changes, migrations, performance, security, and operational concerns.

Be specific and opinionated. Give your best plan, not multiple options.
```

Do not mention model providers or model names in prompts sent to external tools.

## Phase 1: Run Independent Planners

For every enabled external model, start the CLI in parallel and write output to `$SESSION_DIR/{model.id}.md`.

Kilo/OpenRouter:

```bash
{MODEL_COMMAND} "$(cat "$SESSION_DIR/prompt.md")" > "$SESSION_DIR/{MODEL_ID}.md" 2>&1
```

Native Qwen:

```bash
qwen {EXTRA_DIRS_FLAGS} --approval-mode plan -p "$(cat "$SESSION_DIR/prompt.md")" -o text > "$SESSION_DIR/{MODEL_ID}.md" 2>&1
```

Retry once if a command exits non-zero or produces an empty file.

While external models run, Codex independently inspects the codebase and writes its own plan to:

```text
$SESSION_DIR/codex.md
```

Do not wait for external models before starting Codex's own plan.

## Phase 2: Collect

Poll output files roughly once per minute.

A model is working if its file is growing or if no file exists yet. A model is stuck only if the file exists and its size has not changed for 10 consecutive checks. After that, wait 3 more minutes, then mark it failed.

Proceed only if available plans, including `codex.md`, meet `min_quorum`.

Show a status table with Codex and every configured model: done, failed, skipped, or stuck.

## Phase 3: Compare

Read all available plans and show:

- consensus elements
- valuable unique insights
- conflicts
- unintended consequences found by any participant
- comparison table with columns for Codex and each participating model

Do not manufacture agreement. Surface real disagreement.

## Phase 4: Synthesize

Write a unified plan to:

```text
$SESSION_DIR/draft.md
```

The draft should:

1. start with high-confidence consensus items
2. include valuable unique ideas with attribution
3. resolve conflicts with explicit reasoning
4. stay within the original task scope
5. include a dedicated unintended-consequences section

## Phase 5: Convergence

Write `$SESSION_DIR/convergence-prompt.md`:

```markdown
Here is a synthesized implementation plan based on multiple independent reviews:

---
{DRAFT PLAN}
---

Review this plan. Start your response with one of:
- APPROVE - if the plan is solid
- CHANGES NEEDED - if something should change

Only raise issues that genuinely affect correctness, feasibility, or quality.
```

Run the convergence prompt against every external model.

Kilo/OpenRouter:

```bash
{MODEL_COMMAND} {MODEL_RESUME_FLAG} "$(cat "$SESSION_DIR/convergence-prompt.md")" > "$SESSION_DIR/{MODEL_ID}-convergence.md" 2>&1
```

Native Qwen:

```bash
qwen -c -p "$(cat "$SESSION_DIR/convergence-prompt.md")" -o text > "$SESSION_DIR/{MODEL_ID}-convergence.md" 2>&1
```

Codex also reviews the draft locally.

Run at most 2 convergence rounds.

Decision logic:

- all approve: final plan
- minor objections: incorporate valid changes and finalize
- major disagreement after round 2: surface unresolved disagreements to the user

## Final Output

Present the final plan with:

- final implementation plan
- what changed from the initial plan
- remaining disagreements, if any
- idea attribution table
- session directory

Attribution table:

```markdown
| Decision / Element | Origin | Type |
|--------------------|--------|------|
| ... | Codex, Kimi K2.6, ... | Consensus / Unique / Convergence fix |
```

## Rules

1. Codex is the lead. Do not call Codex/GPT as an external reviewer.
2. Use `~/.codex/consensus.json`, not `~/.claude/consensus.json`, unless explicitly asked to inspect Claude config.
3. No model/provider names inside prompts sent to external tools.
4. No code execution by external models beyond read-only inspection, planning, and review.
5. Never use `--yolo`.
6. Use config-driven model lists only.
7. Keep all artifacts in `SESSION_DIR` for auditability.
