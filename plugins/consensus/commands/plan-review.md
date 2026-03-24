---
name: plan-review
description: "Multi-model plan review — AI models independently plan, then converge on the best approach"
---

# Multi-Model Plan Review

Get independent implementation plans from multiple AI models (Claude + configured external models), compare them, and converge on the strongest plan through structured synthesis and approval rounds.

## Compaction Resilience

This is a long-running command. Context compaction may erase in-memory state mid-run.

- **Check resume**: Read `data/scratch/active-progress-consensus-plan-review.md` — if it exists, is <24h old, and the Command field matches `consensus:plan-review`, skip to the first unchecked goal
- **Write progress**: Create the progress file with the goals template below before starting
- **Save incrementally**: Write/append to `$SESSION_DIR` files after each phase, not at the end
- **Context budget**: ~80K tokens (N external model outputs × ~8K + Claude's plan + synthesis)

### Goals Template

```
# consensus:plan-review — {TASK}
Started: [timestamp]
Status: IN_PROGRESS
Command: consensus:plan-review
SESSION_DIR: {SESSION_DIR path}
TTL: 24h

## Goals
- [ ] Phase 1 — Setup: load config, write task prompt, copy plan file if needed, create team
- [ ] Phase 2 — Spawn planners: launch teammate agents + write Claude's plan
- [ ] Phase 3 — Collect plans: wait for all models to send their plans
- [ ] Phase 4 — Analyze & compare: build comparison table, identify consensus approach
- [ ] Phase 5 — Synthesize: draft unified implementation plan
- [ ] Phase 6 — Convergence: send draft to all models, collect APPROVE/CHANGES NEEDED
- [ ] Phase 7 — Write final plan with attribution table, cleanup team

## Progress
- [HH:MM] Starting execution...
```

### Checkpoints
- After Phase 2: Claude's plan written to `$SESSION_DIR/claude.md`
- After Phase 3: All model plans on disk at `$SESSION_DIR/{model.id}.md`
- After Phase 5: Draft plan at `$SESSION_DIR/draft.md`
- After Phase 6: Convergence responses at `$SESSION_DIR/{model.id}-convergence.md`

## Input

`$ARGUMENTS` = the user's original task description / request.

If `$ARGUMENTS` is empty:
1. Check for an existing plan file: `ls -t ~/.claude/plans/*.md | head -1`
2. If a plan file exists, read it and use its content as the task context. This plan was built interactively with the user and contains the task description, codebase context, and a draft approach.
3. If no plan file exists, ask the user: "What task should the panel plan? I need your task description verbatim."

## Step 0: Load Configuration

1. Read `~/.claude/consensus.json` (user override) using the Read tool
2. If not found, read the plugin's `consensus.config.json` from the plugin directory (same directory as the `commands/` folder — i.e., the parent directory of this command file)
3. If neither exists: **ABORT** with: "No config found. Run `/consensus-setup` first to configure your models."
4. Parse the JSON. Filter models where `enabled` is `true` — store as `MODELS` array.
5. Set `MIN_QUORUM` to the **exact value** of the `min_quorum` field from the config JSON. Do NOT calculate or override this value. Use the number the user chose during setup. Only if `min_quorum` is literally missing from the JSON or is not a valid integer >= 2, then fall back to `floor((len(MODELS)+1)/2) + 1`.
6. Validate: `models` must be a non-empty array. Each model must have non-empty `id`, `name`, `command`, and `resume_flag` strings — skip models with missing fields with a warning.
7. If the JSON is malformed (parse error), warn the user and **ABORT**: "Config file is malformed. Run `/consensus-setup` to regenerate."

## Step 0.5: Preflight Checks

Source the API key (targeted — only export `OPENROUTER_API_KEY`):
```bash
[ -f ~/.claude/.env ] && export OPENROUTER_API_KEY=$(grep '^OPENROUTER_API_KEY=' ~/.claude/.env | cut -d= -f2- | tr -d '"')
```

For each model in `MODELS`, verify CLI availability:
- Commands starting with `kilo` -> check: `command -v kilo` AND `[ -n "$OPENROUTER_API_KEY" ]`
- Commands starting with `codex` -> check: `command -v codex`
- Commands starting with `gemini` -> check: `command -v gemini`
- Commands starting with `qwen` -> check: `command -v qwen`

Run all checks in parallel. Remove unavailable models from `MODELS` with a warning for each:
```
Warning: Skipping {model.name} — {reason: "kilo CLI not found" / "OPENROUTER_API_KEY not set" / "codex CLI not found"}
```

Count available models + 1 (Claude) = `TOTAL_PARTICIPANTS`.

If `TOTAL_PARTICIPANTS < MIN_QUORUM`:
**ABORT**: "Only {TOTAL_PARTICIPANTS} models available but quorum requires {MIN_QUORUM}. Run `/consensus-setup` to reconfigure."

Report:
```
Panel: Claude + {comma-separated list of available model names} ({TOTAL_PARTICIPANTS} total, quorum={MIN_QUORUM})
```

## Step 1: Create Session Directory & Write Prompt

```bash
SESSION_DIR=$(mktemp -d /tmp/plan-review-XXXXXX)
```

Print the path so the user knows where artifacts live.

Write the shared planning prompt to `$SESSION_DIR/prompt.md`.

**If input came from `$ARGUMENTS` (task description):**

```markdown
I need to accomplish the following task:

---
{USER'S ORIGINAL MESSAGE — COPIED VERBATIM, NO CHANGES}
---

You have full access to the codebase at `{REPO_DIR}`. Explore it — read relevant files, understand existing patterns and architecture, and base your plan on the actual implementation.

Create a detailed implementation plan. Include:
1. **Files to modify/create** — specific paths
2. **Step-by-step approach** — ordered implementation steps with code-level details
3. **Key decisions** — architectural choices and why
4. **Edge cases** — what could go wrong and how to handle it
5. **Verification** — how to confirm the implementation works

Be specific and opinionated. Give your best plan, not multiple options.
```

**If input came from an existing plan file:**

First, copy the plan file into the repo root so all models can access it:
```bash
cp {ABSOLUTE PATH TO PLAN FILE} $REPO_DIR/.consensus-draft.md
```

Then write the prompt referencing `.consensus-draft.md` by its in-repo path:

```markdown
I need to accomplish a task. A draft plan with task context and codebase analysis exists at:

`.consensus-draft.md` (in the repo root)

Read that file for the full task context and draft approach. You may agree or disagree with the draft — develop your own best plan.

You have full access to the codebase at `{REPO_DIR}`. Explore it — read relevant files, understand existing patterns and architecture, and base your plan on the actual implementation.

Create a detailed implementation plan. Include:
1. **Files to modify/create** — specific paths
2. **Step-by-step approach** — ordered implementation steps with code-level details
3. **Key decisions** — architectural choices and why
4. **Edge cases** — what could go wrong and how to handle it
5. **Verification** — how to confirm the implementation works

Be specific and opinionated. Give your best plan, not multiple options.
```

Clean up after the session: `rm -f $REPO_DIR/.consensus-draft.md`

**Critical rules:**
- Do NOT embed large content in the prompt — all models are CLI tools with direct codebase access; they read files themselves
- If using `$ARGUMENTS`, the task description is short so include it in the prompt verbatim
- If using a plan file, copy it to `.consensus-draft.md` in repo root and reference it by that relative path
- Do NOT mention Claude, Anthropic, OpenAI, Google, Moonshot, xAI, or any AI model name
- Frame as first-person: "I need to accomplish..."

## Step 2: Create Team & Tasks

```
TeamCreate:
  team_name: "plan-review"
  description: "Multi-model plan review"
```

For each model in `MODELS`, create a task using TaskCreate:
- Subject: `"Get {model.name} plan"`
- Description: `"Run {model.name} via CLI and collect implementation plan"`

## Step 3: Spawn All Teammates + Write Claude's Plan

For each model in `MODELS`, spawn a teammate in parallel using the Task tool:

```
Task:
  name: "{model.id}-reviewer"
  team_name: "plan-review"
  subagent_type: "general-purpose"
  model: "sonnet"
  prompt: <see TEAMMATE TEMPLATE below, with variables substituted>
```

**While teammates work**, independently create Claude's own plan using codebase knowledge — Explore agents, Grep, Read, Glob as needed. Write your plan to `$SESSION_DIR/claude.md`.

**Do not wait for teammates before starting Claude's plan.** Work in parallel.

**Expected duration:** External CLI models typically take 3-10 minutes to explore the codebase and produce output. Some models may take longer on complex codebases. This is completely normal — these models almost never fail. Do NOT check on teammates, send messages, or assume failure. Just wait for their SendMessage.

### TEAMMATE TEMPLATE

For each model, substitute `{MODEL_ID}`, `{MODEL_NAME}`, `{MODEL_COMMAND}`, `{MODEL_RESUME_FLAG}`, and `{SESSION_DIR}` into this template:

```
You are {MODEL_ID}-reviewer on the plan-review team. You run {MODEL_NAME} via CLI to get a plan, then participate in convergence rounds.

SESSION_DIR={SESSION_DIR}

## Phase 1: Get Plan

1. Claim your task "Get {MODEL_NAME} plan" via TaskUpdate (set status to in_progress)
2. Source API key (only the key needed, not all env vars):
   [ -f ~/.claude/.env ] && export OPENROUTER_API_KEY=$(grep '^OPENROUTER_API_KEY=' ~/.claude/.env | cut -d= -f2- | tr -d '"')
3. Run the CLI command to get the plan. Use the correct invocation for your CLI type:

   **If `{MODEL_COMMAND}` starts with `codex`:**
   codex exec -s read-only -o $SESSION_DIR/{MODEL_ID}.md - < $SESSION_DIR/prompt.md

   **If `{MODEL_COMMAND}` starts with `gemini`:**
   gemini -p "$(cat $SESSION_DIR/prompt.md)" --approval-mode plan > $SESSION_DIR/{MODEL_ID}.md 2>&1

   **If `{MODEL_COMMAND}` starts with `qwen`:**
   qwen --approval-mode plan -p "$(cat $SESSION_DIR/prompt.md)" -o text > $SESSION_DIR/{MODEL_ID}.md 2>&1

   **Otherwise (Kilo/OpenRouter — default):**
   {MODEL_COMMAND} "$(cat $SESSION_DIR/prompt.md)" > $SESSION_DIR/{MODEL_ID}.md 2>&1

   If it fails or produces empty output, retry ONCE.

4. Read $SESSION_DIR/{MODEL_ID}.md
5. Strip ANSI escape codes and noise
6. Send the clean plan text to the lead via SendMessage (type: "message", recipient: lead name)
7. Mark task completed via TaskUpdate

## Phase 2: Convergence (wait for lead's message)

After sending the plan, WAIT. The lead will send you a convergence prompt. When you receive it:

1. Write the convergence prompt content to $SESSION_DIR/convergence-prompt-{MODEL_ID}.md
2. Run the resume command for your CLI type:

   **If `{MODEL_COMMAND}` starts with `codex`:**
   codex exec resume --last - < $SESSION_DIR/convergence-prompt-{MODEL_ID}.md > $SESSION_DIR/{MODEL_ID}-convergence.md 2>&1

   **If `{MODEL_COMMAND}` starts with `gemini`:**
   gemini --resume latest -p "$(cat $SESSION_DIR/convergence-prompt-{MODEL_ID}.md)" --approval-mode plan > $SESSION_DIR/{MODEL_ID}-convergence.md 2>&1

   **If `{MODEL_COMMAND}` starts with `qwen`:**
   qwen -c -p "$(cat $SESSION_DIR/convergence-prompt-{MODEL_ID}.md)" -o text > $SESSION_DIR/{MODEL_ID}-convergence.md 2>&1

   **Otherwise (Kilo/OpenRouter — default):**
   {MODEL_COMMAND} {MODEL_RESUME_FLAG} "$(cat $SESSION_DIR/convergence-prompt-{MODEL_ID}.md)" > $SESSION_DIR/{MODEL_ID}-convergence.md 2>&1

3. Read the output, clean it
4. Send it to the lead via SendMessage. The response should start with APPROVE or CHANGES NEEDED.

If the lead sends another convergence round, repeat Phase 2.
Wait for a shutdown_request from the lead before exiting.
```

---

## Step 4: Collect Plans

Complete Claude's plan. Then use the following polling protocol to wait for all teammates:

**Polling-based wait loop:**
1. Every ~1 minute, check each pending teammate's output file size:
   `stat -f%z $SESSION_DIR/{model.id}.md 2>/dev/null || echo 0`
2. Track the file size. If it's growing (or the file doesn't exist yet because the model is still exploring) — the model is working. Keep waiting.
3. A teammate is ONLY considered stuck if:
   - Their output file exists AND
   - Its size has not changed for 10 consecutive checks (10 minutes)
4. If a teammate appears stuck after 10 minutes of no file growth, send them a check-in message: "Are you still working? Send me your current output if you have any."
5. Wait another 3 minutes after check-in before giving up on that teammate.
6. DO NOT proceed to Step 5 until every teammate has either sent their result via SendMessage or been declared stuck per the above protocol.

Report to user (dynamically built from `MODELS`):

```
## Plan Collection: {N}/{TOTAL_PARTICIPANTS} Plans Received

- Claude: done/failed
- {model.name}: done/failed (reason)
- ... (one line per model in MODELS)
```

**Proceed if >= MIN_QUORUM plans available** (including Claude's). If fewer than MIN_QUORUM, abort with error listing which models failed and why.

## Step 5: Analyze & Compare

Read all available plans and present a structured comparison:

```
## Panel Results ({N}/{TOTAL_PARTICIPANTS} Plans Received)

### Consensus (all/most models agree)
- {Element}: Agreed by {list of models}
- ...

### Unique Insights
- {Model}: {Insight} — incorporating because {reason} / skipping because {reason}
- ...

### Conflicts
- {Topic}: {Model A} says {X}, {Model B} says {Y}
  -> Recommendation: {which approach and why}
- ...

### Comparison Table
| Aspect | Claude | {model1.name} | {model2.name} | ... |
|--------|--------|---------------|---------------|-----|
| Files changed | ... | ... | ... | ... |
| Key approach | ... | ... | ... | ... |
| Complexity | ... | ... | ... | ... |
| Unique contribution | ... | ... | ... | ... |
```

Build the comparison table columns dynamically from `["Claude"] + [m.name for m in MODELS]`.

## Step 6: Draft Synthesized Plan

Build a single unified plan:

1. Start with **consensus elements** (highest confidence — all/most models agree)
2. Incorporate **valuable unique insights** from individual models (with reasoning for inclusion)
3. For **conflicts**, pick the best approach with explicit reasoning
4. Organize into a clear, implementable plan

Write the draft to `$SESSION_DIR/draft.md`.

## Step 7: Convergence Round — Get Agreement

Write the convergence prompt to `$SESSION_DIR/convergence-prompt.md`:

```markdown
Here is a synthesized implementation plan based on multiple reviews:

---
{DRAFT PLAN FROM $SESSION_DIR/draft.md}
---

Review this plan. Start your response with one of:
- APPROVE — if the plan is solid
- CHANGES NEEDED — if something should change

Then explain your reasoning. Only raise issues that genuinely affect correctness or quality.
```

Send the convergence prompt to all teammates (every model in `MODELS`) via SendMessage. Each teammate will run their model's CLI with resume/context and report back.

Claude also reviews the draft independently.

## Step 8: Evaluate Convergence

Collect all convergence responses from teammate messages. Classify each:

- **APPROVE** — model approves the plan as-is
- **CHANGES NEEDED** — model has objections, extract the specific issues

Decision logic:
- **All approve** -> Draft becomes the final plan. Done.
- **Minor objections** -> Incorporate valid ones into the draft. Note rejected objections with reasoning. This is the final plan.
- **Major disagreements** -> If this is round 1, update the draft and run one more convergence round (back to Step 7). If round 2, present remaining disagreements to the user and let them decide. Mark as "user override."

**Hard limit: 2 convergence rounds.** After round 2, remaining disagreements go to the user.

Show the user:
```
## Convergence Round {N}

### Approvals
- {Model}: APPROVE — {brief reasoning}

### Objections
- {Model}: {objection} — Incorporated / Rejected because {reason}

### Status: {Converged | Running Round 2 | Escalating to user}
```

## Step 9: Write Final Plan

Once converged (or user resolved disagreements):

- If in **plan mode**, update the plan file with the final plan.
- If **not in plan mode**, present the final plan directly to the user.

Include a **provenance footer** and an **idea attribution table** (dynamically built from participating models):

```
---

## Idea Attribution

| Decision / Element | Origin | Type |
|--------------------|--------|------|
| {specific idea from the plan} | {model name(s)} | Consensus / Unique / Convergence fix |
| {another idea} | {model name(s)} | ... |

*Reviewed by {TOTAL_PARTICIPANTS} models: Claude, {comma-separated model names from MODELS}. Convergence: {N} round(s). {any user overrides noted}*
```

**Rules for the attribution table:**
- Every key decision, architectural choice, and non-obvious element must have a row
- "Consensus" = 3+ models proposed the same thing independently
- "Unique" = only 1-2 models proposed it and it was incorporated
- "Convergence fix" = came from convergence round feedback
- Be specific — not "error handling" but "Use retry with exponential backoff on API calls"
- If an idea was rejected, do NOT include it

## Step 10: Cleanup

Send `shutdown_request` to all teammates (every model in `MODELS`). Wait for approvals.

Delete the team:
```
TeamDelete
```

If in plan mode, attempt to call `EnterPlanMode` so the user can review the final plan. If `EnterPlanMode` is unavailable or fails, present the final plan directly to the user instead.

On failure: preserve `$SESSION_DIR` for debugging and tell the user where files are.

## Rules

1. **No model names in prompts.** Never mention any AI model name in prompts sent to external tools. Frame as first-person.
2. **CLI modes:** External models use their configured command with resume flag for convergence. Prompt passed as `"$(cat file.md)"`.
3. **Models have codebase access.** Do NOT embed large content in prompts. Short task descriptions go in the prompt; plan files are referenced by path. Models explore the code themselves.
4. **Show your work.** User sees comparison table, consensus/conflicts, and convergence results at every stage.
5. **Don't manufacture consensus.** Surface real disagreements clearly.
6. **Stay on scope.** No scope creep beyond the original task.
7. **Team-based execution.** Use TeamCreate with dynamically spawned teammates. Each teammate runs one external model and communicates via SendMessage.
8. **Dynamic quorum.** Use `MIN_QUORUM` from config. Abort if fewer than `MIN_QUORUM` plans available (including Claude).
9. **Return to plan mode.** After final plan + cleanup, call `EnterPlanMode` for user review (with fallback to direct presentation).
10. **Convergence through messaging.** Lead sends draft to teammates, they run their model and report back. Max 2 rounds.
11. **Be patient with teammates — they almost never fail.** External CLI models (Codex, Gemini, Kilo) take time to explore the codebase but almost always finish successfully. Follow this activity-based patience protocol:
    - **Poll output files** every ~1 minute using `stat -f%z $SESSION_DIR/{model.id}.md 2>/dev/null || echo 0` to check file size.
    - **Growing file (or no file yet)** = the model is working. Keep waiting.
    - **A teammate is ONLY considered stuck if**: their output file exists AND its size has not changed for **10 consecutive checks** (10 minutes of zero growth).
    - If stuck after 10 minutes, send a check-in message: "Are you still working? Send me your current output if you have any." Wait another 3 minutes before giving up on that teammate.
    - **Idle notifications are completely normal** and must be ignored — they are NOT signals of failure or completion.
    - **NEVER send shutdown requests during Phase 1** (plan collection). Only an explicit SendMessage from the teammate counts as "done."
12. **Never use `--yolo`.** The goal is plans, not code execution.
13. **Config-driven.** All model references come from config files. No hardcoded model names, commands, or counts in this command file.
14. **Targeted API key sourcing.** Only export `OPENROUTER_API_KEY` from `~/.claude/.env`, never export all variables.
