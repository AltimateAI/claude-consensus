---
name: consensus-setup
description: "Configure which AI models to use for multi-model consensus reviews"
---

# Consensus Setup Wizard

Interactive setup to configure which AI models participate in `/code-review` and `/plan-review`.

## Step 1: Check for Existing Config

Check if `~/.claude/consensus.json` exists using the Read tool.

If it exists, show the user what's currently configured (list enabled models and quorum), then ask:

```
AskUserQuestion:
  question: "An existing consensus config was found. What would you like to do?"
  header: "Config"
  options:
    - label: "Reconfigure"
      description: "Start fresh — replace the existing config with new settings"
    - label: "Keep current"
      description: "Exit setup and keep the current configuration"
```

If user chooses "Keep current", print the current config summary and stop.

## Step 2: Auto-Detect CLIs

Run all of these checks in parallel via Bash:

```bash
command -v kilo && echo "KILO_OK" || echo "KILO_MISSING"
```

```bash
command -v codex && echo "CODEX_OK" || echo "CODEX_MISSING"
```

```bash
command -v gemini && echo "GEMINI_OK" || echo "GEMINI_MISSING"
```

```bash
command -v qwen && echo "QWEN_OK" || echo "QWEN_MISSING"
```

Also check for an existing OpenRouter API key (verify it's non-empty):

```bash
[ -f ~/.claude/.env ] && grep -q 'OPENROUTER_API_KEY=.\+' ~/.claude/.env && echo "KEY_FOUND" || echo "KEY_MISSING"
```

Report findings to the user:

```
## CLI Detection Results

- Kilo CLI: {installed / not found}
- Codex CLI: {installed / not found}
- Gemini CLI: {installed / not found}
- Qwen CLI: {installed / not found}
- OpenRouter API key: {found in ~/.claude/.env / not found}
```

## Step 3: Provider Choice

```
AskUserQuestion:
  question: "How do you want to connect to external models?"
  header: "Provider"
  options:
    - label: "OpenRouter (Recommended)"
      description: "1 API key, all 9 models via Kilo CLI. Simplest setup."
    - label: "Native CLIs"
      description: "Use codex, gemini, qwen CLIs directly where available. Requires each CLI installed separately."
    - label: "Both"
      description: "Mix and match — use native CLIs where available, OpenRouter/Kilo for the rest."
```

## Step 4: API Key Setup

**If OpenRouter was selected (or "Both"):**

Check `~/.claude/.env` for a non-empty `OPENROUTER_API_KEY`:

```bash
[ -f ~/.claude/.env ] && grep -q 'OPENROUTER_API_KEY=.\+' ~/.claude/.env && echo "KEY_SET" || echo "NOT_SET"
```

If the key exists and is non-empty, tell the user: "OpenRouter API key found in `~/.claude/.env`."

If the key is missing or empty, ask the user:

```
AskUserQuestion:
  question: "Enter your OpenRouter API key (get one at https://openrouter.ai/keys):"
  header: "API Key"
  options:
    - label: "I'll paste it"
      description: "Provide the key now and I'll save it to ~/.claude/.env"
    - label: "Skip for now"
      description: "I'll set OPENROUTER_API_KEY manually later"
```

If user provides a key, update `~/.claude/.env` idempotently:
1. Read the existing file (if it exists)
2. If `OPENROUTER_API_KEY` line exists, replace it
3. If it doesn't exist, append it
4. Preserve all other lines unchanged
5. Write the updated file

**If "Native CLIs" only:** skip this step.

## Step 5: Model Selection

Build the model list based on provider choice and detected CLIs.

The 9 supported models and their CLI mappings:

| Model ID | Name | OpenRouter (Kilo) | Native CLI |
|----------|------|-------------------|------------|
| `gpt` | GPT 5.4 Codex | `kilo run -m openrouter/openai/gpt-5.4-codex --auto` | `codex` (if codex CLI installed) |
| `gemini` | Gemini 3.1 Pro | `kilo run -m openrouter/google/gemini-3.1-pro-preview --auto` | `gemini` (if gemini CLI installed) |
| `kimi` | Kimi K2.5 | `kilo run -m openrouter/moonshotai/kimi-k2.5 --auto` | OpenRouter only |
| `grok` | Grok 4 | `kilo run -m openrouter/x-ai/grok-4 --auto` | OpenRouter only |
| `minimax` | MiniMax M2.7 | `kilo run -m openrouter/minimax/minimax-m2.7 --auto` | OpenRouter only |
| `glm5` | GLM-5 | `kilo run -m openrouter/z-ai/glm-5 --auto` | OpenRouter only |
| `qwen` | Qwen 3.5 Plus | `kilo run -m openrouter/qwen/qwen3.5-plus-02-15 --auto` | `qwen` (if qwen CLI installed) |
| `nemotron` | Nemotron 120B | `kilo run -m openrouter/nvidia/nemotron-3-super-120b-a12b --auto` | OpenRouter only |
| `mimo` | MiMo V2 Pro | `kilo run -m openrouter/xiaomi/mimo-v2-pro --auto` | OpenRouter only |

**Note on native CLIs**: For `codex`, `gemini`, and `qwen`, set the config's `command` field to just `codex`, `gemini`, or `qwen`. The teammate template in the review/plan commands detects these and uses the correct native invocation patterns automatically (e.g., `codex exec -s read-only` for reviews, `codex exec resume --last` for convergence, `gemini -p` for reviews, `gemini --resume latest` for convergence, `qwen --approval-mode plan -p` with `-o text` for reviews, `qwen -c -p` for convergence). The `resume_flag` field is ignored for native CLIs.

Determine which models are available:
- **OpenRouter path**: All 9 available if `kilo` installed + API key set
- **Native path**: Only `gpt` (if codex installed), `gemini` (if gemini installed), and `qwen` (if qwen installed)
- **Both path**: Native CLI where available, OpenRouter/Kilo for the rest

```
AskUserQuestion:
  question: "Which models do you want to enable? (Claude is always included)"
  header: "Models"
  multiSelect: true
  options:
    - label: "GPT 5.4 Codex"
      description: "{available via OpenRouter / available via codex CLI / not available}"
    - label: "Gemini 3.1 Pro"
      description: "{available via OpenRouter / available via gemini CLI / not available}"
    - label: "Kimi K2.5"
      description: "{available via OpenRouter / not available}"
    - label: "Grok 4"
      description: "{available via OpenRouter / not available}"
    - label: "MiniMax M2.7"
      description: "{available via OpenRouter / not available}"
    - label: "GLM-5"
      description: "{available via OpenRouter / not available}"
    - label: "Qwen 3.5 Plus"
      description: "{available via OpenRouter / available via qwen CLI / not available}"
    - label: "Nemotron 120B"
      description: "{available via OpenRouter / not available}"
    - label: "MiMo V2 Pro"
      description: "{available via OpenRouter / not available}"
```

(Show all 9 models. Mark unavailable ones clearly. Pre-select available ones.)

**Enforce: at least 1 external model must be selected.** If user selects none, tell them: "At least 1 external model is required for consensus reviews."

## Step 6: Quorum Selection

Calculate the recommended quorum: `floor(total_participants / 2) + 1` where `total_participants = enabled_externals + 1` (Claude).

Example: 4 external models + Claude = 5 total -> quorum = 3.

```
AskUserQuestion:
  question: "What should the minimum quorum be? (models that must respond for a valid review)"
  header: "Quorum"
  options:
    - label: "{recommended} (Recommended)"
      description: "Strict majority of {total} participants (Claude + {N} external models)"
    - label: "2"
      description: "Minimum viable — Claude + 1 external model"
    - label: "{total}"
      description: "Unanimous — all {total} participants must respond"
```

Constraints:
- Minimum: 2 (Claude + 1 external)
- Maximum: total participants
- Recommended: `floor(total/2) + 1`

## Step 7: Write Config

Build the config JSON based on selections from Steps 3-6.

For each of the 9 models:
- If the model was selected: set `enabled: true` and populate `command`/`resume_flag` based on provider choice
- If the model was NOT selected: set `enabled: false` with the default OpenRouter command (so users can re-enable later)

For enabled models:
- If user chose "Native CLIs" or "Both" AND the native CLI is detected, use the native command + native resume flag
- Otherwise use the OpenRouter/Kilo command + `-c` resume flag

Write the config to `~/.claude/consensus.json` using the Write tool:

```json
{
  "version": "1.0.0",
  "models": [
    {
      "id": "{model_id}",
      "name": "{model_name}",
      "command": "{selected command}",
      "resume_flag": "{selected resume flag}",
      "enabled": true
    },
    {
      "id": "{disabled_model_id}",
      "name": "{disabled_model_name}",
      "command": "{default openrouter command}",
      "resume_flag": "-c",
      "enabled": false
    }
  ],
  "min_quorum": {selected_quorum}
}
```

All 9 models are always written to the config — enabled or disabled.

## Step 8: Smoke Test

For each enabled model, run a quick test:

```bash
[ -f ~/.claude/.env ] && export OPENROUTER_API_KEY=$(grep '^OPENROUTER_API_KEY=' ~/.claude/.env | cut -d= -f2- | tr -d '"')
{model.command} "Reply with exactly: PONG" 2>&1 | head -20
```

Check if the output contains "PONG" (case-insensitive).

- **Pass**: Report success for that model
- **Fail**: Set `enabled: false` for that model in the config, warn the user

Report results:

```
## Smoke Test Results

- GPT 5.4 Codex: PASS
- Gemini 3.1 Pro: PASS
- Kimi K2.5: FAIL — {error or empty output}
- ...
```

**After all tests**: Count remaining enabled models + Claude. If total < min_quorum:

**HARD STOP**: Print:
```
Cannot finalize config: only {N} models available but quorum requires {min_quorum}.
Either lower the quorum or fix the failing models, then re-run /consensus-setup.
```

Do NOT finalize the config in this case. Delete `~/.claude/consensus.json` and abort.

If quorum is still met after disabling failed models, rewrite `~/.claude/consensus.json` with updated enabled/disabled states.

## Step 8.5: Performance Tracking

Ask the user if they want to enable automatic performance tracking:

```
AskUserQuestion:
  question: "Enable performance tracking? Records which models contribute unique findings so you can identify and prune underperformers."
  header: "Performance Tracking"
  options:
    - label: "Yes (Recommended)"
      description: "Append metrics to ~/.claude/multi-model-performance.json after each run. View with /consensus:performance."
    - label: "No"
      description: "Skip performance tracking"
```

If **Yes**:
1. Set `"performance_tracking": true` in `~/.claude/consensus.json` (add the field to the config written in Step 7)
2. If `~/.claude/multi-model-performance.json` does not exist, create it:
   ```json
   {
     "version": 1,
     "runs": []
   }
   ```
3. Report: "Performance tracking enabled. View results anytime with `/consensus:performance`."

If **No**:
1. Set `"performance_tracking": false` in `~/.claude/consensus.json`
2. Report: "Performance tracking disabled. You can enable it later by re-running `/consensus-setup`."

## Step 9: Summary

Print the final summary:

```
## Consensus Setup Complete

**Config saved to:** `~/.claude/consensus.json`

**Enabled models ({N} + Claude = {total} participants):**
{list of enabled model names}

**Quorum:** {min_quorum} of {total}

**Performance tracking:** {enabled / disabled}

**Next steps:**
- `/code-review [target]` — multi-model code review
- `/plan-review [task]` — multi-model plan review
- `/review [target]` — multi-model document or general review
- `/consensus:performance` — view model performance data (if tracking enabled)
- `/consensus-setup` — reconfigure models anytime
```

## Rules

1. **9 fixed models only.** Do not offer custom model configuration. The wizard supports exactly the 9 models listed above.
2. **Idempotent .env updates.** When writing API keys, preserve all existing keys in the file. Only add/update the `OPENROUTER_API_KEY` line.
3. **Native CLI support for codex, gemini, and qwen.** When a user selects native CLIs, set the config's `command` field to `codex`, `gemini`, or `qwen`. The teammate template in the review/plan commands handles the full invocation patterns automatically. All other models use OpenRouter/Kilo only.
4. **OpenRouter is the recommended path.** 1 key = 9 models. Emphasize this as the simplest setup.
5. **Enforce minimum 1 external model.** Claude alone is not a consensus.
6. **Hard-stop on quorum failure.** Never finalize a config that can't meet its own quorum.
7. **User config location is `~/.claude/consensus.json`.** The plugin default at `plugins/consensus/consensus.config.json` is the fallback only.
8. **Write all 9 models to config.** Include disabled models with `enabled: false` so users can re-enable them later without re-running setup.
9. **Targeted API key sourcing.** Only export `OPENROUTER_API_KEY` from `~/.claude/.env`, never export all variables.
