---
name: code-review
description: "Multi-model code review — AI models independently review code, then converge on findings"
---

# Multi-Model Code Review

Get independent code reviews from multiple AI models (Claude + configured external models), compare findings, and converge on a unified review through structured synthesis and agreement rounds.

## Compaction Resilience

This is a long-running command. Context compaction may erase in-memory state mid-run.

- **Check resume**: Glob `data/scratch/active-progress-consensus-code-review-*.md` — find any with Status `IN_PROGRESS` and < 24h old. If found, read its SESSION_DIR and skip to first unchecked goal. If Status is `COMPLETED` or `FAILED`, ignore it.
- **Write progress**: After creating SESSION_DIR, create a unique progress file: `data/scratch/active-progress-consensus-code-review-{SESSION_ID}.md` (where SESSION_ID is the random suffix from SESSION_DIR, e.g. `X4f2kL`)
- **Mark done**: Set Status to `COMPLETED` at end of successful run, or `FAILED` on abort
- **Save incrementally**: Write/append to `$SESSION_DIR` files after each phase, not at the end

### Goals Template

```
# consensus:code-review — {TARGET}
Started: [timestamp]
Status: IN_PROGRESS
Command: consensus:code-review
SESSION_DIR: {SESSION_DIR path}
TTL: 24h

## Goals
- [ ] Phase 1 — Setup: load config, determine review target, write prompt, create team
- [ ] Phase 2 — Spawn reviewers: launch teammate agents + write Claude's code review
- [ ] Phase 3 — Collect reviews: wait for all models to send their reviews
- [ ] Phase 4 — Analyze & compare: build comparison table, identify consensus issues
- [ ] Phase 5 — Synthesize: draft unified code review ordered by severity
- [ ] Phase 6 — Convergence: send draft to all models, collect APPROVE/CHANGES NEEDED
- [ ] Phase 7 — Present final review with attribution table, cleanup team

## Progress
- [HH:MM] Starting execution...
```

### Checkpoints
- After Phase 2: Claude's review written to `$SESSION_DIR/claude.md`
- After Phase 3: All model reviews on disk at `$SESSION_DIR/{model.id}.md`
- After Phase 5: Draft review at `$SESSION_DIR/draft.md`
- After Phase 6: Convergence responses at `$SESSION_DIR/{model.id}-convergence.md`

## Input

`$ARGUMENTS` = what to review. Examples: `PR #123`, `staged changes`, `last 3 commits`, `app/service/foo.py`, or a description of what changed.

### Additional Directories

If the review spans multiple repositories, the user can pass `--dirs` to give all models access to additional directories:

```
/consensus:code-review PR #42 --dirs /path/to/frontend,/path/to/backend
```

**Parsing `--dirs`:**
1. Extract the `--dirs` value from `$ARGUMENTS` (everything after `--dirs` up to the next `--` flag or end of string)
2. Split on commas to get a list of absolute paths -> store as `EXTRA_DIRS` array
3. Remove the `--dirs ...` portion from `$ARGUMENTS` so it doesn't pollute the review target
4. Validate each path exists with `test -d`. Warn and skip any that don't exist.
5. If no `--dirs` flag is present, `EXTRA_DIRS` is empty (no additional directories)

Store `EXTRA_DIRS` for use in Step 1 (prompt) and Step 3 (teammate template).

If `$ARGUMENTS` is empty:
1. Run `git diff HEAD` and `git diff --cached` to capture unstaged and staged changes
2. If both are empty, run `git diff main...HEAD` to get all changes on the current branch vs main
3. If still empty, ask the user: "What code should the panel review? Give me a PR number, file paths, or describe the changes."

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

**Check CodeRabbit availability:**
```bash
command -v coderabbit && echo "CODERABBIT_OK"
```
If available, set `CODERABBIT_AVAILABLE=true`. CodeRabbit is a supplementary static analysis reviewer — it does NOT count toward quorum and does NOT participate in convergence. Its findings are incorporated during synthesis.

Count available models + 1 (Claude) = `TOTAL_PARTICIPANTS`. If CodeRabbit is available, add 1 to `TOTAL_PARTICIPANTS` for reporting (but NOT for quorum calculation).

If `TOTAL_PARTICIPANTS (excluding CodeRabbit) < MIN_QUORUM`:
**ABORT**: "Only {TOTAL_PARTICIPANTS} models available but quorum requires {MIN_QUORUM}. Run `/consensus-setup` to reconfigure."

Report:
```
Panel: Claude + {comma-separated list of available model names}{+ CodeRabbit if available} ({TOTAL_PARTICIPANTS} total, quorum={MIN_QUORUM})
```

## Step 1: Create Session Directory & Write Prompt

```bash
SESSION_DIR=$(mktemp -d /tmp/code-review-XXXXXX)
```

Print the path so the user knows where artifacts live.

**Determine the review target** based on `$ARGUMENTS` and figure out the appropriate git command:
- "the latest commit" / "last commit" -> `git diff HEAD~1..HEAD`
- "last N commits" -> `git diff HEAD~N..HEAD`
- PR number -> `gh pr diff {number}`
- "staged changes" -> `git diff --cached`
- File paths -> review those specific files
- Branch diff -> `git diff main...HEAD`

Write the shared review prompt to `$SESSION_DIR/prompt.md`:

```markdown
I need a thorough code review. The review target is: {DESCRIPTION — e.g., "the latest commit dfe20acf"}

To see the changes, run: `{GIT COMMAND — e.g., git diff HEAD~1..HEAD}`

You have full access to the codebase. Explore it — read changed files, related tests, imports, and existing patterns before reviewing. Do NOT review the diff in isolation.

{IF EXTRA_DIRS is non-empty, add this paragraph:}
Additional repositories are available for context at these paths — explore them if the changes reference or depend on code there:
{for each dir in EXTRA_DIRS:}
- `{dir}`
{end for}

For each issue found, provide:

1. **Severity**: CRITICAL / MAJOR / MINOR / NIT
2. **Category**: Bug, Security, Performance, Logic Error, Code Quality, Design, Testing, Documentation
3. **Location**: File path and line number(s)
4. **Description**: What the issue is and why it matters
5. **Suggestion**: How to fix it (with code if applicable)

Also provide:
- **Overall assessment**: Is this code ready to merge? What's the overall quality?
- **Positive observations**: What's done well (good patterns, clean code, etc.)
- **Missing tests**: Any untested paths or edge cases

Be specific and opinionated. Flag real issues, not style preferences. Prioritize correctness and security over cosmetics.
```

**Critical rules:**
- Do NOT embed diffs in the prompt — all models are CLI tools with direct codebase access; they run git commands themselves
- Do NOT mention Claude, Anthropic, OpenAI, Google, Moonshot, xAI, or any AI model name
- Frame as first-person: "I need a thorough code review..."

## Step 2: Create Team & Tasks

```
TeamCreate:
  team_name: "code-review"
  description: "Multi-model code review"
```

For each model in `MODELS`, create a task using TaskCreate:
- Subject: `"Get {model.name} review"`
- Description: `"Run {model.name} via CLI and collect code review"`

## Step 3: Spawn All Teammates + Write Claude's Review

For each model in `MODELS`, spawn a teammate in parallel using the Task tool:

```
Task:
  name: "{model.id}-reviewer"
  team_name: "code-review"
  subagent_type: "general-purpose"
  model: "sonnet"
  prompt: <see TEAMMATE TEMPLATE below, with variables substituted>
```

**While teammates work**, do two things in parallel:

1. **Run CodeRabbit** (if `CODERABBIT_AVAILABLE`):
   ```bash
   coderabbit review --plain --base {BASE_BRANCH or BASE_COMMIT} > $SESSION_DIR/coderabbit.md 2>&1
   ```
   - Use the same base reference as the review target (e.g., `--base main` for branch diffs, `--base-commit HEAD~N` for commit ranges)
   - CodeRabbit runs fast (typically 30-60 seconds) and writes structured findings directly
   - If it fails or returns empty output, set `CODERABBIT_AVAILABLE=false` and continue without it

2. **Write Claude's own review** using codebase knowledge — Read the changed files, related tests, understand patterns. Write your review to `$SESSION_DIR/claude.md`.

**Do not wait for teammates before starting Claude's review or CodeRabbit.** Work in parallel.

**Expected duration:** External CLI models typically take 3-10 minutes to explore the codebase and produce output. Some models may take longer on complex codebases. This is completely normal — these models almost never fail. Do NOT check on teammates, send messages, or assume failure. Just wait for their SendMessage.

### TEAMMATE TEMPLATE

For each model, substitute `{MODEL_ID}`, `{MODEL_NAME}`, `{MODEL_COMMAND}`, `{MODEL_RESUME_FLAG}`, `{SESSION_DIR}`, and `{EXTRA_DIRS_FLAGS}` into this template.

**Building `{EXTRA_DIRS_FLAGS}`** — if `EXTRA_DIRS` is non-empty, build per-CLI flags:
- For commands starting with `codex`: `--add-dir /path1 --add-dir /path2` (one `--add-dir` per directory)
- For commands starting with `gemini`: `--include-directories /path1,/path2` (comma-separated)
- For commands starting with `qwen`: `--include-directories /path1,/path2` (comma-separated)
- For commands starting with `kilo`: empty string (kilo has no flag — the paths are already in the prompt)

If `EXTRA_DIRS` is empty, `{EXTRA_DIRS_FLAGS}` is an empty string for all CLIs.

```
You are {MODEL_ID}-reviewer on the code-review team. You run {MODEL_NAME} via CLI to get a code review, then participate in convergence rounds.

SESSION_DIR={SESSION_DIR}

## Phase 1: Get Review

1. Claim your task "Get {MODEL_NAME} review" via TaskUpdate (set status to in_progress)
2. Source API key (only the key needed, not all env vars):
   [ -f ~/.claude/.env ] && export OPENROUTER_API_KEY=$(grep '^OPENROUTER_API_KEY=' ~/.claude/.env | cut -d= -f2- | tr -d '"')
3. Run the CLI command to get the review. Use the correct invocation for your CLI type:

   **If `{MODEL_COMMAND}` starts with `codex`:**
   codex exec -s read-only {EXTRA_DIRS_FLAGS} -o $SESSION_DIR/{MODEL_ID}.md - < $SESSION_DIR/prompt.md

   **If `{MODEL_COMMAND}` starts with `gemini`:**
   gemini {EXTRA_DIRS_FLAGS} -p "$(cat $SESSION_DIR/prompt.md)" > $SESSION_DIR/{MODEL_ID}.md 2>&1

   **If `{MODEL_COMMAND}` starts with `qwen`:**
   qwen {EXTRA_DIRS_FLAGS} --approval-mode plan -p "$(cat $SESSION_DIR/prompt.md)" -o text > $SESSION_DIR/{MODEL_ID}.md 2>&1

   **Otherwise (Kilo/OpenRouter — default):**
   {MODEL_COMMAND} "$(cat $SESSION_DIR/prompt.md)" > $SESSION_DIR/{MODEL_ID}.md 2>&1

   If it fails or produces empty output, retry ONCE.

4. Read $SESSION_DIR/{MODEL_ID}.md
5. Strip ANSI escape codes and noise
6. Send the clean review text to the lead via SendMessage (type: "message", recipient: lead name)
7. Mark task completed via TaskUpdate

## Phase 2: Convergence (wait for lead's message)

After sending the review, WAIT. The lead will send you a convergence prompt. When you receive it:

1. Write the convergence prompt content to $SESSION_DIR/convergence-prompt-{MODEL_ID}.md
2. Run the resume command for your CLI type:

   **If `{MODEL_COMMAND}` starts with `codex`:**
   codex exec resume --last - < $SESSION_DIR/convergence-prompt-{MODEL_ID}.md > $SESSION_DIR/{MODEL_ID}-convergence.md 2>&1

   **If `{MODEL_COMMAND}` starts with `gemini`:**
   gemini --resume latest -p "$(cat $SESSION_DIR/convergence-prompt-{MODEL_ID}.md)" > $SESSION_DIR/{MODEL_ID}-convergence.md 2>&1

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

## Step 4: Collect Reviews

Complete Claude's review. Then use the following polling protocol to wait for all teammates:

**Polling-based wait loop:**
1. Every ~1 minute, check each pending teammate's output file size:
   `wc -c < $SESSION_DIR/{model.id}.md 2>/dev/null || echo 0`
2. Track the file size. If it's growing (or the file doesn't exist yet because the model is still exploring) — the model is working. Keep waiting.
3. A teammate is ONLY considered stuck if:
   - Their output file exists AND
   - Its size has not changed for 10 consecutive checks (10 minutes)
4. If a teammate appears stuck after 10 minutes of no file growth, send them a check-in message: "Are you still working? Send me your current output if you have any."
5. Wait another 3 minutes after check-in before giving up on that teammate.
6. DO NOT proceed to Step 5 until every teammate has either sent their result via SendMessage or been declared stuck per the above protocol.

Report to user (dynamically built from `MODELS`):

```
## Review Collection: {N}/{TOTAL_PARTICIPANTS} Reviews Received

- Claude: done/failed
- CodeRabbit: done/skipped/failed (only if CODERABBIT_AVAILABLE)
- {model.name}: done/failed (reason)
- ... (one line per model in MODELS)
```

**Proceed if >= MIN_QUORUM reviews available** (including Claude's). If fewer than MIN_QUORUM, abort with error listing which models failed and why.

## Step 5: Analyze & Compare

Read all available reviews — including CodeRabbit's output at `$SESSION_DIR/coderabbit.md` if it ran — and present a structured comparison.

**CodeRabbit findings**: CodeRabbit produces structured findings with file paths, line numbers, types (potential_issue, refactor_suggestion), and suggested fixes. Parse these and include them alongside the AI model reviews. CodeRabbit findings are treated as an additional signal — they carry weight like any other reviewer but CodeRabbit does NOT participate in convergence rounds.

```
## Review Results ({N}/{TOTAL_PARTICIPANTS} Reviews Received)

### Consensus Issues (multiple models flagged the same thing)
- {Issue}: Flagged by {list of models} — Severity: {highest severity among them}
- ...

### Unique Findings (only 1-2 models caught this)
- {Model}: {Finding} — incorporating because {reason} / skipping because {reason}
- ...

### Disagreements
- {Topic}: {Model A} says {X is a bug}, {Model B} says {X is fine}
  -> Recommendation: {which assessment is correct and why}
- ...

### Comparison Table
| Issue | Claude | {model1.name} | {model2.name} | ... |
|-------|--------|---------------|---------------|-----|
| {issue 1} | CRITICAL | CRITICAL | - | ... |
| {issue 2} | - | MINOR | MINOR | ... |
| Overall verdict | {pass/fail} | ... | ... | ... |
```

Build the comparison table columns dynamically from `["Claude"] + (["CodeRabbit"] if CODERABBIT_AVAILABLE) + [m.name for m in MODELS]`.

## Step 6: Draft Synthesized Review

Build a single unified code review:

1. Start with **consensus issues** (highest confidence — multiple models flagged the same thing)
2. Incorporate **valuable unique findings** from individual models (with reasoning for inclusion)
3. For **disagreements**, determine the correct assessment with explicit reasoning
4. Organize by severity: CRITICAL > MAJOR > MINOR > NIT
5. Include positive observations that multiple models noted

Write the draft to `$SESSION_DIR/draft.md`.

Format:

```markdown
## Code Review Summary

**Verdict**: {APPROVE / REQUEST CHANGES / BLOCK}
**Critical Issues**: {count}
**Major Issues**: {count}
**Minor Issues**: {count}

### Critical Issues
1. **{Title}** ({Category}) — {file}:{line}
   {Description}
   **Fix**: {suggestion with code}
   _Flagged by: {models}_

### Major Issues
...

### Minor Issues
...

### Positive Observations
- {What's done well}

### Missing Tests
- {Untested paths}
```

## Step 7: Convergence Round — Get Agreement

Write the convergence prompt to `$SESSION_DIR/convergence-prompt.md`:

```markdown
Here is a synthesized code review based on multiple independent reviews:

---
{DRAFT REVIEW FROM $SESSION_DIR/draft.md}
---

Review this synthesized assessment. Start your response with one of:
- APPROVE — if the review findings are accurate and complete
- CHANGES NEEDED — if findings are incorrect, missing, or mis-categorized

Then explain your reasoning. Only raise issues about the review's accuracy, not new code issues you didn't mention before.
```

Send the convergence prompt to all teammates (every model in `MODELS`) via SendMessage. Each teammate will run their model's CLI with resume/context and report back.

Claude also reviews the draft independently.

## Step 8: Evaluate Convergence

Collect all convergence responses from teammate messages. Classify each:

- **APPROVE** — model agrees with the synthesized review
- **CHANGES NEEDED** — model disagrees with specific findings

Decision logic:
- **All approve** -> Draft becomes the final review. Done.
- **Minor objections** -> Incorporate valid ones (e.g., severity re-classification, missed context). Note rejected objections with reasoning. This is the final review.
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

## Step 9: Present Final Review

Once converged (or user resolved disagreements):

Present the final review directly to the user.

Include a **finding attribution table** (dynamically built from participating models):

```
---

## Finding Attribution

| Issue | Origin | Type |
|-------|--------|------|
| {specific issue from the review} | {model name(s)} | Consensus / Unique / Convergence fix |
| {another issue} | {model name(s)} | ... |

*Reviewed by {TOTAL_PARTICIPANTS} participants: Claude, {comma-separated model names from MODELS}{, CodeRabbit (static analysis) if it ran}. Convergence: {N} round(s). {any user overrides noted}*
```

**Rules for the attribution table:**
- Every issue in the final review must have a row
- "Consensus" = 3+ models flagged the same issue independently
- "Unique" = only 1-2 models caught it and it was incorporated
- "Convergence fix" = severity changed or issue added/removed during convergence
- Be specific — not "security issue" but "SQL injection in user_query parameter at line 42"
- If a finding was rejected, do NOT include it

## Step 10: Cleanup

Send `shutdown_request` to all teammates (every model in `MODELS`). Wait for approvals.

Delete the team:
```
TeamDelete
```

On failure: preserve `$SESSION_DIR` for debugging and tell the user where files are.

## Step 11: Record Performance Metrics (if enabled)

Check the `performance_tracking` field from the config loaded in Step 0. If it is `true`, execute this step. If `false` or missing, skip entirely.

### 11a: Gather Data

For each model that participated (Claude + all models in `MODELS`, + CodeRabbit if it ran):

1. **Response status**: Check `$SESSION_DIR/{model-id}.md`
   - Exists and > 0 bytes → `"yes"`
   - Exists but 0 bytes → `"failed"`
   - Does not exist → `"timeout"`

2. **Output size**: `wc -c < $SESSION_DIR/{model-id}.md 2>/dev/null || echo 0`

3. **Convergence vote**: Read `$SESSION_DIR/{model-id}-convergence.md`. Find first occurrence of `APPROVE` or `CHANGES NEEDED`. If file missing or neither found → `null`. (CodeRabbit has no convergence vote — always `null`.)

4. **Attribution counts from `$SESSION_DIR/draft.md`**: Find all `_Flagged by:` lines. For each line:
   - Parse model names between `_Flagged by:` and `_`
   - Normalize names to model IDs using config model names and IDs. Also: Claude→claude, CodeRabbit→coderabbit.
   - If 1-2 models listed → each gets +1 `unique_findings`
   - If 3+ models listed → each gets +1 `consensus_findings`
   - Count total `_Flagged by:` lines as `total_findings`

### 11b: Build Run Record

```json
{
  "id": "{SESSION_ID from SESSION_DIR path suffix}",
  "timestamp": "{current UTC ISO 8601}",
  "command": "consensus:code-review",
  "target": "{short description of review target}",
  "review_type": "CODE_REVIEW",
  "total_findings": "{count}",
  "models": {
    "{model-id}": {
      "responded": "yes|failed|timeout",
      "unique_findings": "{N}",
      "consensus_findings": "{N}",
      "convergence_vote": "APPROVE|CHANGES NEEDED|null",
      "output_bytes": "{N}"
    }
  }
}
```

### 11c: Append to JSON

1. Read `~/.claude/multi-model-performance.json`. If it does not exist, create it with `{"version": 1, "runs": []}`.
2. Parse JSON
3. Check if `id` already exists in `runs[]` — if so, skip (dedup)
4. Append run record to `runs[]`
5. Write updated JSON back

Report: `"Performance tracked: {total_findings} findings across {M} models. {len(runs)} total runs recorded."`

## Rules

1. **No model names in prompts.** Never mention any AI model name in prompts sent to external tools. Frame as first-person.
2. **CLI modes:** External models use their configured command with resume flag for convergence. Prompt passed as `"$(cat file.md)"`.
3. **Models have codebase access.** Do NOT embed diffs in prompts. The prompt tells models what to review and what git command to run — they explore the code themselves.
4. **Show your work.** User sees comparison table, consensus/disagreements, and convergence results at every stage.
5. **Don't manufacture consensus.** Surface real disagreements about findings clearly.
6. **Severity matters.** A bug found by one model is still a bug. Don't dismiss unique findings just because only one model caught them — evaluate on merit.
7. **Team-based execution.** Use TeamCreate with dynamically spawned teammates. Each teammate runs one external model and communicates via SendMessage.
8. **Dynamic quorum.** Use `MIN_QUORUM` from config. Abort if fewer than `MIN_QUORUM` reviews available (including Claude).
9. **Convergence through messaging.** Lead sends draft to teammates, they run their model and report back. Max 2 rounds.
10. **No plan mode.** Code reviews are presented directly, not written to plan files.
11. **Be patient with teammates — they almost never fail.** External CLI models (Codex, Gemini, Kilo) take time to explore the codebase but almost always finish successfully. Follow this activity-based patience protocol:
    - **Poll output files** every ~1 minute using `wc -c < $SESSION_DIR/{model.id}.md 2>/dev/null || echo 0` to check file size.
    - **Growing file (or no file yet)** = the model is working. Keep waiting.
    - **A teammate is ONLY considered stuck if**: their output file exists AND its size has not changed for **10 consecutive checks** (10 minutes of zero growth).
    - If stuck after 10 minutes, send a check-in message: "Are you still working? Send me your current output if you have any." Wait another 3 minutes before giving up on that teammate.
    - **Idle notifications are completely normal** and must be ignored — they are NOT signals of failure or completion.
    - **NEVER send shutdown requests during Phase 1** (review collection). Only an explicit SendMessage from the teammate counts as "done."
12. **Config-driven.** All model references come from config files. No hardcoded model names, commands, or counts in this command file.
13. **Targeted API key sourcing.** Only export `OPENROUTER_API_KEY` from `~/.claude/.env`, never export all variables.
