# claude-consensus

**Multi-model code review, plan review, and general review for Claude Code and Codex**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Plugin-blueviolet)](https://claude.ai/claude-code)

Multiple AI models independently review code, implementation plans, documents, or technical proposals, then converge through structured synthesis and approval rounds. The lead host participates directly: Claude leads in Claude Code, and Codex leads in Codex.

## Quick Start

### Claude Code

Tell Claude:

> "Set up claude-consensus: https://github.com/AltimateAI/claude-consensus"

Manual install:

```bash
/plugin marketplace add AltimateAI/claude-consensus
/plugin install consensus
/consensus-setup
```

Then use:

```bash
/code-review "staged changes"
/plan-review "Add caching to the API layer"
/consensus:review docs/architecture.md
```

### Codex

From a local clone, add this repo marketplace to Codex and install the `consensus` plugin:

```bash
git clone https://github.com/AltimateAI/claude-consensus.git
```

Use the repo marketplace at:

```text
.agents/plugins/marketplace.json
```

Then configure and run:

```text
consensus-setup
consensus-code-review staged changes
consensus-plan-review Add caching to the API layer
consensus-review docs/architecture.md
```

Alias skills are also provided for `code-review`, `plan-review`, `plan-reviwe`, and `claude-consensus`.

## Prerequisites

| Requirement | Required? | Notes |
|-------------|-----------|-------|
| Claude Code or Codex | Yes | The host environment determines the lead reviewer |
| Kilo CLI | Recommended | Routes OpenRouter models with one API key |
| OpenRouter API key | Recommended | Required for Kilo/OpenRouter models |
| Native CLIs | Optional | Claude supports `codex`, `gemini`, and `qwen`; Codex supports non-Codex native CLIs |
| CodeRabbit CLI | Optional | Supplementary static analysis for code reviews |

**Minimal setup**: the host lead plus 1 external model is enough for consensus reviews.

## Supported Panels

| Model | Provider | OpenRouter ID | Claude native | Codex external |
|-------|----------|---------------|---------------|----------------|
| Claude | Anthropic | built-in | lead | not used |
| Codex / GPT | OpenAI | `openai/gpt-5.4-codex` | `codex` | lead, not external |
| Gemini 3.1 Pro | Google | `google/gemini-3.1-pro-preview` | `gemini` | `gemini` |
| Kimi K2.6 | Moonshot | `moonshotai/kimi-k2.6` | Kilo | Kilo |
| Grok 4.20 | xAI | `x-ai/grok-4.20-beta` | Kilo | Kilo, disabled by default |
| MiniMax M2.7 | MiniMax | `minimax/minimax-m2.7` | Kilo | Kilo |
| GLM-5.1 | Zhipu AI | `zai-coding-plan/glm-5.1` | Kilo | Kilo |
| Qwen 3.6 Plus | Alibaba | `qwen/qwen3.6-plus` | `qwen` or Kilo | Kilo by default |
| MiMo V2 Pro | Xiaomi | `xiaomi/mimo-v2-pro` | Kilo | Kilo |
| DeepSeek V4 Pro | DeepSeek | `deepseek/deepseek-v4-pro` | Kilo | Kilo |

Codex intentionally excludes Codex/GPT from the external panel because Codex is already the lead reviewer.

## Configuration

| Host | User config | Default config | Env file |
|------|-------------|----------------|----------|
| Claude Code | `~/.claude/consensus.json` | `plugins/consensus/consensus.config.json` | `~/.claude/.env` |
| Codex | `~/.codex/consensus.json` | `plugins/consensus/consensus.codex.config.json` | `~/.codex/.env` first, then `~/.claude/.env` read-only fallback |

Claude setup can smoke-test live model calls. Codex setup does not run live smoke tests unless you ask, because they consume credits.

Example Codex config:

```json
{
  "version": "1.6.0",
  "lead": {
    "id": "codex",
    "name": "Codex",
    "excluded_model_ids": ["gpt", "codex"]
  },
  "models": [
    {
      "id": "gemini",
      "name": "Gemini 3.1 Pro",
      "command": "gemini",
      "resume_flag": "",
      "enabled": true
    },
    {
      "id": "kimi",
      "name": "Kimi K2.6",
      "command": "kilo run -m openrouter/moonshotai/kimi-k2.6 --auto",
      "resume_flag": "-c",
      "enabled": true
    }
  ],
  "min_quorum": 3
}
```

## Commands And Skills

| Host | Command / skill | Purpose |
|------|-----------------|---------|
| Claude | `/consensus-setup` | Configure models, API keys, and quorum |
| Claude | `/code-review [target]` | Multi-model code review with convergence |
| Claude | `/plan-review [task]` | Multi-model implementation planning |
| Claude | `/consensus:review [target]` | General document/topic review |
| Codex | `consensus-setup` | Configure the Codex consensus panel |
| Codex | `consensus-code-review [target]` | Codex-led code review with convergence |
| Codex | `consensus-plan-review [task]` | Codex-led implementation plan review |
| Codex | `consensus-review [target]` | Codex-led document/topic review |

## How It Works

```text
Host lead  Model 1  Model 2  Model N
    |         |        |        |
    v         v        v        v
Independent review or planning
    |_________|________|________|
              |
              v
Synthesis: consensus, conflicts, comparison table
              |
              v
Convergence: APPROVE / CHANGES NEEDED, max 2 rounds
              |
              v
Final result with attribution
```

- **Quorum**: configurable per host.
- **Graceful degradation**: unavailable models are skipped if quorum still holds.
- **Session artifacts**: saved in `/tmp/` for debugging and auditability.
- **Independent reviews**: each model starts from the same prompt with no cross-contamination.
- **Codex isolation**: Codex uses separate config and never writes Claude user config.

## Contributing

Contributions are welcome. Please read the [Contributing Guide](CONTRIBUTING.md) before submitting a PR.

This project follows the [Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md).

## License

MIT License. Copyright (c) 2026 [Altimate AI](https://altimate.ai/).
