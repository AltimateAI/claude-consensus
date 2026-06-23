# Codex Consensus Setup

This is the Codex-native setup flow for the Consensus plugin. It configures the external model panel used by `consensus-code-review`, `consensus-plan-review`, and `consensus-review`.

## Files

- User config: `~/.codex/consensus.json`
- Codex default config: `plugins/consensus/consensus.codex.config.json`
- OpenRouter key lookup: `~/.codex/.env` first, then `~/.claude/.env` as a read-only fallback

Never write `~/.claude/consensus.json` from this setup command.

## Rules

1. Codex is the lead reviewer in Codex sessions.
2. Do not include GPT/Codex as an external panelist.
3. For now, do not add Claude as an external panelist.
4. Use Kilo/OpenRouter for OpenRouter-hosted models and native non-Codex CLIs only.
5. Preserve user model preferences unless the user explicitly asks to replace them.
6. Do not smoke-test live model calls unless the user asks, because that consumes credits.

## Step 1: Inspect Current State

Check:

```bash
test -f ~/.codex/consensus.json && jq . ~/.codex/consensus.json
command -v kilo
command -v qwen
test -f ~/.codex/.env && grep -q '^OPENROUTER_API_KEY=.\+' ~/.codex/.env
test -f ~/.claude/.env && grep -q '^OPENROUTER_API_KEY=.\+' ~/.claude/.env
```

Report:

- whether `~/.codex/consensus.json` exists and parses
- enabled external models
- configured quorum
- Kilo and Qwen availability
- whether an OpenRouter key exists in `~/.codex/.env` or fallback `~/.claude/.env`

If the existing config is valid and the user only asked to inspect, stop after the summary.

## Step 2: Provider Defaults

Use these model mappings:

| ID | Name | Default command | Native alternative |
|----|------|-----------------|--------------------|
| `kimi` | Kimi K2.6 | `kilo run -m openrouter/moonshotai/kimi-k2.6 --auto` | none |
| `grok` | Grok 4.20 | `kilo run -m openrouter/x-ai/grok-4.20-beta --auto` | none |
| `minimax` | MiniMax M2.7 | `kilo run -m openrouter/minimax/minimax-m2.7 --auto` | none |
| `qwen` | Qwen 3.6 Plus | `kilo run -m openrouter/qwen/qwen3.6-plus --auto` | optional `qwen`, disabled by default unless user asks |
| `mimo` | MiMo V2 Pro | `kilo run -m openrouter/xiaomi/mimo-v2-pro --auto` | none |
| `deepseek` | DeepSeek V4 Pro | `kilo run -m openrouter/deepseek/deepseek-v4-pro --auto` | none |

Codex setup intentionally omits the Claude plugin's `gpt` model because Codex/GPT is the lead in Codex sessions.

Recommended defaults:

- enable the current stable non-Codex panel: Kimi, MiniMax, Qwen, MiMo, and DeepSeek
- leave Grok disabled by default unless the user explicitly enables it
- use Kilo/OpenRouter for all enabled models
- set quorum to 6 for the default panel, meaning all 5 enabled externals plus Codex must respond

For custom panels, keep quorum within these bounds:

```text
recommended = floor((enabled_external_models + 1) / 2) + 1
minimum = 2
maximum = enabled_external_models + 1
```

## Step 3: Key Handling

If Kilo/OpenRouter models are selected and no OpenRouter key is found, ask the user for the key or tell them to set it later.

When saving a key, write only to `~/.codex/.env`:

1. create the file if needed
2. replace an existing `OPENROUTER_API_KEY=` line if present
3. otherwise append it
4. preserve all other lines unchanged

Never copy the key into repo files or print it back to the user.

## Step 4: Write Config

Write `~/.codex/consensus.json` with this shape:

```json
{
  "version": "1.6.0",
  "lead": {
    "id": "codex",
    "name": "Codex",
    "excluded_model_ids": ["gpt", "codex"],
    "note": "Codex is the lead reviewer in Codex sessions, so do not invoke Codex/GPT as an external panelist."
  },
  "models": [
    {
      "id": "kimi",
      "name": "Kimi K2.6",
      "command": "kilo run -m openrouter/moonshotai/kimi-k2.6 --auto",
      "resume_flag": "-c",
      "enabled": true
    }
  ],
  "min_quorum": 2,
  "performance_tracking": false
}
```

Include all 6 non-Codex external models, with `enabled` reflecting the user's selected panel.

Use the selected quorum. The default Codex panel uses `min_quorum: 6`.

For enabled Kilo models, use `resume_flag: "-c"`.

For native Qwen, only use `command: "qwen"` if the user explicitly selects native Qwen; otherwise prefer Kilo/OpenRouter.

## Step 5: Optional Smoke Test

Only run this if the user explicitly asks for live validation:

```bash
if [ -f ~/.codex/.env ] && grep -q '^OPENROUTER_API_KEY=.\+' ~/.codex/.env; then
  export OPENROUTER_API_KEY="$(grep '^OPENROUTER_API_KEY=' ~/.codex/.env | cut -d= -f2- | tr -d '"')"
elif [ -f ~/.claude/.env ] && grep -q '^OPENROUTER_API_KEY=.\+' ~/.claude/.env; then
  export OPENROUTER_API_KEY="$(grep '^OPENROUTER_API_KEY=' ~/.claude/.env | cut -d= -f2- | tr -d '"')"
fi
```

For each enabled model, run a tiny prompt asking it to reply `PONG`.

If a model fails, disable it only after reporting the failure and confirming that quorum still holds. If quorum would fail, leave the prior valid config in place and tell the user what needs to be fixed.

## Final Output

Summarize:

- config path
- enabled external models
- lead model: Codex
- quorum
- skipped or unavailable models
- whether live smoke tests were skipped or run
