---
name: review
description: "Multi-model review — AI models independently review any document or general topic, then converge on findings"
---

# Multi-Model Review

Get independent reviews from multiple AI models (Claude + configured external models) on documents, plans, specs, or any free-text topic. Compare findings and converge on a unified review through structured synthesis and agreement rounds.

For code reviews, use `/consensus:code-review`. For implementation plans, use `/consensus:plan-review`. This command covers everything else.

## Compaction Resilience

This is a long-running command. Context compaction may erase in-memory state mid-run.

- **Check resume**: Glob `data/scratch/active-progress-consensus-review-*.md` — find any with Status `IN_PROGRESS` and < 24h old. If found, read its SESSION_DIR and skip to first unchecked goal. If Status is `COMPLETED` or `FAILED`, ignore it.
- **Write progress**: After creating SESSION_DIR, create a unique progress file: `data/scratch/active-progress-consensus-review-{SESSION_ID}.md` (where SESSION_ID is the random suffix from SESSION_DIR, e.g. `X4f2kL`)
- **Mark done**: Set Status to `COMPLETED` at end of successful run, or `FAILED` on abort
- **Save incrementally**: Write/append to `$SESSION_DIR` files after each phase, not at the end

### Goals Template

```
# consensus:review — {TARGET}
Started: [timestamp]
Status: IN_PROGRESS
Command: consensus:review
SESSION_DIR: {SESSION_DIR path}
TTL: 24h

## Goals
- [ ] Phase 1 — Setup: load config, detect review type, write prompt, create team
- [ ] Phase 2 — Spawn reviewers: launch teammate agents + write Claude's review
- [ ] Phase 3 — Collect reviews: wait for all models to send their reviews
- [ ] Phase 4 — Analyze & compare: build comparison table, identify consensus issues
- [ ] Phase 5 — Synthesize: draft unified review (type-specific format)
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

`$ARGUMENTS` = what to review. Examples: `~/.claude/plans/my-plan.md`, `the plan`, `docs/architecture.md`, or a free-text description like `"should we use Redis or Postgres for this queue?"`.

### Input Detection

Auto-detect the review type from `$ARGUMENTS`:

| Priority | Pattern | Type |
|----------|---------|------|
| 1 | File path to `.md`, `.txt`, `.rst`, `.adoc` file (must exist via `test -f`), or `the plan` / `latest plan` shorthand | DOCUMENT_REVIEW |
| 2 | Everything else (free-text description) | GENERAL_REVIEW |

**File existence check**: For DOCUMENT_REVIEW, verify the file exists with `test -f {path}`. If it doesn't exist, treat as GENERAL_REVIEW with a note: "File not found at `{path}` — treating as general review."

**`the plan` / `latest plan` shorthand**: Resolves to the most recent file from `ls -t ~/.claude/plans/*.md | head -1`.

**Empty input fallback** (deterministic order):
1. Check `ls -t ~/.claude/plans/*.md | head -1` — if exists, use DOCUMENT_REVIEW on that file
2. Ask user: "What should the panel review? Give me a file path or describe what to review."

After detection:
- Print: `"Detected review type: {TYPE} — reviewing {description}"`
- Store type for downstream steps: `echo "{TYPE}" > $SESSION_DIR/type.txt`

### Review Type Labels

| Type | Label |
|------|-------|
| DOCUMENT_REVIEW | document review |
| GENERAL_REVIEW | assessment |

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
SESSION_DIR=$(mktemp -d /tmp/review-XXXXXX)
```

Print the path so the user knows where artifacts live.

Write the shared review prompt to `$SESSION_DIR/prompt.md` based on the detected review type:

---

### DOCUMENT_REVIEW Prompt

If the document is outside the repo, first copy it into the repo root so all models can access it:
```bash
cp {ABSOLUTE PATH TO DOCUMENT} $REPO_DIR/.consensus-draft.md
```

```markdown
I need a thorough review of this document/plan.

Read this document completely: `{FILE_PATH or .consensus-draft.md if copied to repo root}`

**Verify claims against the actual codebase** — do not trust the document blindly. Check that referenced files, functions, APIs, and patterns actually exist and behave as described.

Evaluate against these 7 criteria:
1. **Accuracy**: Are factual claims about the codebase correct? Do referenced files/functions exist?
2. **Completeness**: Are there missing steps, unaddressed edge cases, or gaps?
3. **Feasibility**: Can this plan be implemented as described? Are there hidden blockers?
4. **Internal Consistency**: Does the document contradict itself?
5. **Actionability**: Are instructions specific enough to execute without guessing?
6. **Risks & Assumptions**: Are assumptions stated? Are risks identified and mitigated?
7. **Structure**: Is the document well-organized and navigable?

For each issue found, provide:
1. **Severity**: CRITICAL / MAJOR / MINOR / NIT
2. **Category**: One of the 7 criteria above
3. **Location**: Section or line reference in the document
4. **Description**: What the issue is and why it matters
5. **Evidence**: What the codebase actually shows vs what the document claims

Also provide:
- **Readiness verdict**: YES (ready to execute) / NEEDS REVISION / NOT READY
- **Factual errors**: Claims that contradict the codebase, with evidence
- **Completeness gaps**: What's missing

Prioritize accuracy and feasibility. Challenge assumptions.
```

Clean up after the session: `rm -f $REPO_DIR/.consensus-draft.md`

---

### GENERAL_REVIEW Prompt

```markdown
I need your assessment of the following:

---
{USER'S DESCRIPTION — COPIED VERBATIM}
---

You have full access to the codebase. Explore relevant files to validate claims and ground your assessment.

Evaluate against these 5 dimensions:
1. **Correctness**: Are the claims, logic, or approach correct?
2. **Completeness**: Is anything missing or overlooked?
3. **Quality**: Is this well-thought-out? Are there better approaches?
4. **Risks**: What could go wrong? What are the downsides?
5. **Alternatives**: Are there better ways to achieve the same goal?

For each finding, provide:
1. **Severity**: CRITICAL / MAJOR / MINOR / NIT
2. **Category**: One of the 5 dimensions above
3. **Description**: What the finding is and why it matters
4. **Suggestion**: How to address it

Also provide:
- **Overall verdict**: SOUND (solid approach) / NEEDS REVISION / FLAWED (fundamental problems)
- **Top 3 improvements**: Highest-impact changes
- **Alternative approaches**: Different ways to achieve the goal, with trade-offs

Be specific and opinionated.
```

---

**Critical rules for all prompt types:**
- Do NOT embed large content in the prompt — reference by path. Models have codebase access.
- Do NOT mention Claude, Anthropic, OpenAI, Google, Moonshot, xAI, or any AI model name
- Frame as first-person: "I need a thorough review..."

## Step 2: Create Team & Tasks

```
TeamCreate:
  team_name: "review"
  description: "Multi-model review"
```

For each model in `MODELS`, create a task using TaskCreate:
- Subject: `"Get {model.name} review"`
- Description: `"Run {model.name} via CLI and collect review"`

## Step 3: Spawn All Teammates + Write Claude's Review

For each model in `MODELS`, spawn a teammate in parallel using the Task tool:

```
Task:
  name: "{model.id}-reviewer"
  team_name: "review"
  subagent_type: "general-purpose"
  model: "sonnet"
  prompt: <see TEAMMATE TEMPLATE below, with variables substituted>
```

**While teammates work**, independently create Claude's own review using codebase knowledge — Read the relevant files, explore context, understand patterns. Write your review to `$SESSION_DIR/claude.md`.

**Do not wait for teammates before starting Claude's review.** Work in parallel.

**Expected duration:** External CLI models typically take 3-10 minutes to explore the codebase and produce output. Some models may take longer on complex codebases. This is completely normal — these models almost never fail. Do NOT check on teammates, send messages, or assume failure. Just wait for their SendMessage.

### TEAMMATE TEMPLATE

For each model, substitute `{MODEL_ID}`, `{MODEL_NAME}`, `{MODEL_COMMAND}`, `{MODEL_RESUME_FLAG}`, and `{SESSION_DIR}` into this template:

```
You are {MODEL_ID}-reviewer on the review team. You run {MODEL_NAME} via CLI to get a review, then participate in convergence rounds.

SESSION_DIR={SESSION_DIR}

## Phase 1: Get Review

1. Claim your task "Get {MODEL_NAME} review" via TaskUpdate (set status to in_progress)
2. Source API key (only the key needed, not all env vars):
   [ -f ~/.claude/.env ] && export OPENROUTER_API_KEY=$(grep '^OPENROUTER_API_KEY=' ~/.claude/.env | cut -d= -f2- | tr -d '"')
3. Run the CLI command to get the review. Use the correct invocation for your CLI type:

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
6. Send the clean review text to the lead via SendMessage (type: "message", recipient: lead name)
7. Mark task completed via TaskUpdate

## Phase 2: Convergence (wait for lead's message)

After sending the review, WAIT. The lead will send you a convergence prompt. When you receive it:

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
- {model.name}: done/failed (reason)
- ... (one line per model in MODELS)
```

**Proceed if >= MIN_QUORUM reviews available** (including Claude's). If fewer than MIN_QUORUM, abort with error listing which models failed and why.

## Step 5: Analyze & Compare

Read all available reviews and present a structured comparison. The comparison table format depends on the review type (read from `$SESSION_DIR/type.txt`):

### Shared Sections (all types)

```
## Review Results ({N}/{TOTAL_PARTICIPANTS} Reviews Received)

### Consensus Issues (multiple models flagged the same thing)
- {Issue}: Flagged by {list of models} — Severity: {highest severity among them}
- ...

### Unique Findings (only 1-2 models caught this)
- {Model}: {Finding} — incorporating because {reason} / skipping because {reason}
- ...

### Disagreements
- {Topic}: {Model A} says {X}, {Model B} says {Y}
  -> Recommendation: {which assessment is correct and why}
- ...
```

### Comparison Table — DOCUMENT_REVIEW

```
| Model | Factual errors | Completeness gaps | Feasibility concerns | Ready? |
|-------|----------------|-------------------|----------------------|--------|
| Claude | {N} | {N} | {N} | YES/REV/NO |
| {model1.name} | ... | ... | ... | ... |
| ... | ... | ... | ... | ... |
```

### Comparison Table — GENERAL_REVIEW

```
| Model | Issues | Risks | Alternatives | Verdict |
|-------|--------|-------|--------------|---------|
| Claude | {N} | {N} | {N} | SOUND/REV/FLAWED |
| {model1.name} | ... | ... | ... | ... |
| ... | ... | ... | ... | ... |
```

Build the comparison table columns dynamically from `["Claude"] + [m.name for m in MODELS]`.

## Step 6: Draft Synthesized Review

Build a single unified review. The synthesis format depends on the review type:

### DOCUMENT_REVIEW Synthesis

1. Start with **consensus issues** (highest confidence — multiple models flagged the same thing)
2. Incorporate **valuable unique findings** from individual models (with reasoning for inclusion)
3. For **disagreements**, determine the correct assessment with explicit reasoning
4. Organize by category: Factual Errors > Completeness Gaps > Feasibility Concerns

```markdown
## Document Review Summary

**Verdict**: {YES / NEEDS REVISION / NOT READY}
**Factual Errors**: {count}
**Completeness Gaps**: {count}
**Feasibility Concerns**: {count}

### Factual Errors
1. **{Claim}** — {document section}
   **Document says**: {what the document claims}
   **Codebase shows**: {what actually exists, with file path evidence}
   _Flagged by: {models}_

### Completeness Gaps
1. **{Missing element}**
   {What's missing and why it matters}
   _Flagged by: {models}_

### Feasibility Concerns
1. **{Concern}**
   {Why this might not work as described}
   _Flagged by: {models}_

### Positive Observations
- {What's done well}
```

### GENERAL_REVIEW Synthesis

1. Start with **consensus issues** (highest confidence — multiple models flagged the same thing)
2. Incorporate **valuable unique findings** from individual models (with reasoning for inclusion)
3. For **disagreements**, determine the correct assessment with explicit reasoning
4. Organize by severity: CRITICAL > MAJOR > MINOR

```markdown
## Assessment Summary

**Verdict**: {SOUND / NEEDS REVISION / FLAWED}
**Critical Issues**: {count}
**Major Issues**: {count}
**Minor Issues**: {count}

### Issues by Severity

#### Critical
...

#### Major
...

#### Minor
...

### Alternative Approaches
1. **{Alternative}**: {description}
   - **Pros**: {advantages}
   - **Cons**: {disadvantages}
   - **Trade-off vs current**: {comparison}

### Top 3 Improvements
1. {Highest-impact change}
2. ...
3. ...

### Positive Observations
- {What's sound}
```

Write the draft to `$SESSION_DIR/draft.md`.

## Step 7: Convergence Round — Get Agreement

Read `$SESSION_DIR/type.txt` to get the review type and its label.

Write the convergence prompt to `$SESSION_DIR/convergence-prompt.md`:

```markdown
Here is a synthesized {REVIEW_TYPE_LABEL} based on multiple independent reviews:

---
{DRAFT REVIEW FROM $SESSION_DIR/draft.md}
---

Review this synthesized {REVIEW_TYPE_LABEL}. Start your response with one of:
- APPROVE — if the findings are accurate and complete
- CHANGES NEEDED — if findings are incorrect, missing, or mis-categorized

Then explain your reasoning. Only raise issues about the review's accuracy, not new issues you didn't mention before.
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

**Review type**: {REVIEW_TYPE} — {description of what was reviewed}

| Issue | Origin | Type |
|-------|--------|------|
| {specific issue from the review} | {model name(s)} | Consensus / Unique / Convergence fix |
| {another issue} | {model name(s)} | ... |

*{REVIEW_TYPE_LABEL} by {TOTAL_PARTICIPANTS} models: Claude, {comma-separated model names from MODELS}. Convergence: {N} round(s). {any user overrides noted}*
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

For DOCUMENT_REVIEW, clean up the copied file:
```bash
rm -f $REPO_DIR/.consensus-draft.md
```

On failure: preserve `$SESSION_DIR` for debugging and tell the user where files are.

## Rules

1. **No model names in prompts.** Never mention any AI model name in prompts sent to external tools. Frame as first-person.
2. **CLI modes:** External models use their configured command with resume flag for convergence. Prompt passed as `"$(cat file.md)"`.
3. **Models have codebase access.** Do NOT embed large content in prompts. Reference files by path — models explore the code themselves.
4. **Show your work.** User sees comparison table, consensus/disagreements, and convergence results at every stage.
5. **Don't manufacture consensus.** Surface real disagreements about findings clearly.
6. **Severity matters.** A finding from one model is still valid. Don't dismiss unique findings just because only one model caught them — evaluate on merit.
7. **Team-based execution.** Use TeamCreate with dynamically spawned teammates. Each teammate runs one external model and communicates via SendMessage.
8. **Dynamic quorum.** Use `MIN_QUORUM` from config. Abort if fewer than `MIN_QUORUM` reviews available (including Claude).
9. **Convergence through messaging.** Lead sends draft to teammates, they run their model and report back. Max 2 rounds.
10. **No plan mode.** Reviews are presented directly, not written to plan files.
11. **Be patient with teammates — they almost never fail.** External CLI models (Codex, Gemini, Kilo) take time to explore the codebase but almost always finish successfully. Follow this activity-based patience protocol:
    - **Poll output files** every ~1 minute using `wc -c < $SESSION_DIR/{model.id}.md 2>/dev/null || echo 0` to check file size.
    - **Growing file (or no file yet)** = the model is working. Keep waiting.
    - **A teammate is ONLY considered stuck if**: their output file exists AND its size has not changed for **10 consecutive checks** (10 minutes of zero growth).
    - If stuck after 10 minutes, send a check-in message: "Are you still working? Send me your current output if you have any." Wait another 3 minutes before giving up on that teammate.
    - **Idle notifications are completely normal** and must be ignored — they are NOT signals of failure or completion.
    - **NEVER send shutdown requests during Phase 1** (review collection). Only an explicit SendMessage from the teammate counts as "done."
12. **Type-aware synthesis.** The review type detected in the Input step drives the prompt template (Step 1), comparison table format (Step 5), synthesis structure (Step 6), convergence label (Step 7), and attribution footer (Step 9). Read `$SESSION_DIR/type.txt` whenever type-specific behavior is needed. Never mix formats across types.
13. **Config-driven.** All model references come from config files. No hardcoded model names, commands, or counts in this command file.
14. **Targeted API key sourcing.** Only export `OPENROUTER_API_KEY` from `~/.claude/.env`, never export all variables.

## Edge Cases

| Edge Case | Handling |
|-----------|----------|
| File path doesn't exist | Check `test -f`. Fall through to GENERAL_REVIEW with note: "File not found — treating as general review." |
| `the plan` but no plan files exist | Ask user: "No plan files found. What should the panel review?" |
| Empty input, no plan files | Ask user with clear prompt. Follow the deterministic fallback order. |
| Existing `.consensus-draft.md` in repo root | Overwrite it (it's a temp file from consensus commands). |
| Team "review" already exists | Detect stale team, warn user, offer cleanup before proceeding. |
| Document is inside the repo | Reference by relative path directly — no need to copy to `.consensus-draft.md`. |
| Document is outside the repo | Copy to `$REPO_DIR/.consensus-draft.md` so all CLI models can access it. |
