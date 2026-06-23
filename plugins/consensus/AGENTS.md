# Consensus Plugin

This plugin supports two hosts:

- Claude Code commands in `commands/`
- Codex skills in `skills/`

Do not modify the command or skill files without understanding the full multi-model execution flow. The same product behavior is implemented differently per host because Claude Code command files can use Claude-native team primitives, while Codex skills run a Codex-led workflow with direct CLI subprocess collection.

## Claude Configuration

Claude commands load model configuration from:

1. `~/.claude/consensus.json` - user config created by `/consensus-setup`
2. `plugins/consensus/consensus.config.json` - Claude plugin defaults

Claude is always the lead participant. Native CLIs (`codex`, `qwen`) are handled as special cases in the command templates.

## Codex Configuration

Codex skills load model configuration from:

1. `~/.codex/consensus.json` - user config created by `consensus-setup`
2. `plugins/consensus/consensus.codex.config.json` - Codex plugin defaults

Codex is always the lead participant. Codex/GPT must not be invoked as an external panelist, even if a user config contains a stale `gpt` or `codex` entry.

Codex setup may read `~/.claude/.env` as a fallback source for `OPENROUTER_API_KEY`, but it must not write `~/.claude/consensus.json`.

## Runtime Rules

At runtime, workflows verify CLI availability for each configured model and skip unavailable ones. If the remaining host lead plus external models do not meet `min_quorum`, the workflow aborts with a clear message.

The final synthesis must preserve:

- independent first-pass reviews or plans
- comparison tables
- consensus and unique findings
- conflicts and unresolved disagreements
- convergence rounds
- attribution tables

## Public Entrypoints

Claude commands:

- `/consensus-setup`
- `/code-review`
- `/plan-review`
- `/consensus:review`

Codex skills:

- `consensus-setup`
- `consensus-code-review`
- `consensus-plan-review`
- `consensus-review`
- aliases: `code-review`, `plan-review`, `plan-reviwe`, `claude-consensus`
