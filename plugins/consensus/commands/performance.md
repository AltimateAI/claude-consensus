---
name: performance
description: "View consolidated multi-model performance data — identify which models contribute and which to prune"
---

# Multi-Model Performance Dashboard

Analyze performance data from `~/.claude/multi-model-performance.json` to show which models contribute unique findings, which are redundant, and overall response reliability.

## Input

`$ARGUMENTS` = optional filter. Examples:
- Empty — show full dashboard
- `last 10` — show only the last 10 runs
- `code-review` — filter to code-review runs only
- `plan-review` — filter to plan-review runs only
- `model gpt` — show detailed stats for a specific model

## Step 1: Load Data

Read `~/.claude/multi-model-performance.json`.

If it does not exist or is empty:
**ABORT**: "No performance data found. Enable tracking via `/consensus-setup` (Step 8.5) or ensure `performance_tracking` is `true` in `~/.claude/consensus.json`."

Parse JSON. Extract `runs[]` array.

If `$ARGUMENTS` contains a filter, apply it:
- `last N` → take only the last N runs (by timestamp)
- `code-review` / `plan-review` / `review` → filter by `command` field
- `model {id}` → skip to Step 4 (single model deep dive)

Report: `"Loaded {len(runs)} runs ({filtered count if filtered})."`

## Step 2: Model Leaderboard

Aggregate across all (filtered) runs. For each model that appears in any run:

1. **Runs participated**: Count of runs where `responded == "yes"`
2. **Response rate**: `runs_participated / total_runs * 100`
3. **Unique findings**: Sum of `unique_findings` across all runs
4. **Consensus findings**: Sum of `consensus_findings` across all runs
5. **Total contributions**: `unique + consensus`
6. **Unique rate**: `unique / total_contributions * 100` (how often this model catches things others miss)
7. **Avg convergence**: Count APPROVE vs CHANGES NEEDED votes

Present as a sorted table (sort by unique findings descending):

```
## Model Leaderboard ({N} runs)

| Model | Runs | Response % | Unique | Consensus | Total | Unique % | Convergence |
|-------|------|-----------|--------|-----------|-------|----------|-------------|
| {model} | {N} | {pct}% | {N} | {N} | {N} | {pct}% | {N}A / {N}C |
| ... | ... | ... | ... | ... | ... | ... | ... |

A = APPROVE, C = CHANGES NEEDED
```

## Step 3: Actionable Insights

Based on the leaderboard, generate insights:

### Prune Candidates
Models with **0 unique findings** across all runs — they never catch anything the other models miss.

```
### Prune Candidates (0 unique findings)
- {model}: {N} runs, 0 unique findings, {N} consensus findings
  → Consider disabling in /consensus-setup to save time and API costs
```

### MVPs
Models with the **highest unique finding rate** — they consistently catch issues others miss.

```
### MVPs (highest unique contribution)
- {model}: {unique_findings} unique findings ({unique_rate}% of their contributions are unique)
```

### Reliability Issues
Models with **response rate < 90%** — they timeout or fail frequently.

```
### Reliability Issues (response rate < 90%)
- {model}: {response_rate}% response rate ({failures} failures, {timeouts} timeouts)
```

### Contrarians
Models that vote **CHANGES NEEDED** more than 50% of the time during convergence — they frequently disagree with the synthesis.

```
### Contrarians (>50% CHANGES NEEDED votes)
- {model}: {changes_needed_count}/{total_votes} convergence rounds resulted in objections
```

## Step 4: Single Model Deep Dive (if `model {id}` filter)

If the user requested a specific model, show detailed per-run history:

```
## {model_name} — Detailed Performance

**Overall:** {runs} runs, {response_rate}% response rate, {unique} unique / {consensus} consensus findings

### Run History
| Run | Date | Command | Target | Unique | Consensus | Vote | Output Size |
|-----|------|---------|--------|--------|-----------|------|-------------|
| {id} | {date} | {cmd} | {target} | {N} | {N} | {vote} | {bytes} |
| ... | ... | ... | ... | ... | ... | ... | ... |

### Trends
- Finding rate trend: {increasing / stable / decreasing}
- Most common contribution type: {unique / consensus}
- Average output size: {N} bytes
```

## Step 5: Recommendations

Based on all data, provide a concrete recommendation:

```
## Recommendations

**Current panel:** Claude + {N} models, quorum={quorum}
**Suggested changes:**
- {Disable {model} — 0 unique findings in {N} runs}
- {Keep {model} — highest unique rate at {N}%}
- {Monitor {model} — response rate dropping ({N}%)}
- {Consider lowering quorum to {N} if you disable {N} models}
```

## Data Location

Performance data is stored at: `~/.claude/multi-model-performance.json`

Each run record contains:
- `id` — session ID
- `timestamp` — UTC ISO 8601
- `command` — which consensus command was used
- `target` — what was reviewed/planned
- `review_type` — CODE_REVIEW, PLAN_REVIEW, DOCUMENT_REVIEW, or GENERAL_REVIEW
- `total_findings` — total number of findings in the synthesized output
- `models` — per-model breakdown with response status, findings, convergence vote, and output size

## Rules

1. **Read-only.** This command never modifies the performance JSON. It only reads and analyzes.
2. **Handle sparse data.** If a model appears in some runs but not others, calculate rates based on runs where it was expected (present in the config at the time).
3. **No minimum run requirement.** Show data even with 1 run, but note: "Limited data — {N} run(s). Recommendations improve with more data."
4. **Be specific.** Name exact models, give exact numbers. No vague "some models are underperforming."
5. **Actionable output.** Every insight must have a clear action: disable, keep, monitor, or reconfigure.
