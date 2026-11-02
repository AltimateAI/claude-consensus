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
| pi CLI | Recommended | Routes OpenRouter models with one API key |
| OpenRouter API key | Recommended | Required for pi/OpenRouter models |
| Native CLIs | Optional | Claude supports `codex`, Antigravity CLI (`agy`), and `qwen`; Codex supports non-Codex native CLIs |
| CodeRabbit CLI | Optional | Supplementary static analysis for code reviews |

**Minimal setup**: the host lead plus 1 external model is enough for consensus reviews.

## Supported Panels

| Model | Provider | OpenRouter ID | Claude native | Codex external |
|-------|----------|---------------|---------------|----------------|
| Claude | Anthropic | built-in | lead | not used |
| Codex / GPT | OpenAI | `openai/gpt-6.1-sol` | `codex` | lead, not external |
| Gemini 4 Argon | Google | `google/gemini-4-argon` | Antigravity CLI (`agy`) | Antigravity CLI (`agy`) |
| Kimi K3.1 Pro | Moonshot | `moonshotai/kimi-k3.1-pro` | pi | pi |
| Grok 4.7 | xAI | `x-ai/grok-4.7` | pi | pi, disabled by default |
| MiniMax M3 | MiniMax | `minimax/minimax-m3` | pi | pi |
| GLM-5.3 | Zhipu AI | `z-ai/glm-5.3` | pi | pi |
| Qwen 3.8 Max | Alibaba | `qwen/qwen3.8-max-0902` | `qwen` or pi | pi by default |
| MiMo V2.6 Pro | Xiaomi | `xiaomi/mimo-v2.6-pro` | pi | pi |
| DeepSeek V4 Pro | DeepSeek | `deepseek/deepseek-v4-pro-0813` | pi | pi |

Codex intentionally excludes Codex/GPT from the external panel because Codex is already the lead reviewer.

Setup can also replace a slot with a newer OpenRouter model, or add extra panelists with their own ids; see the setup command for the rules.

## Configuration

| Host | User config | Default config | Env file |
|------|-------------|----------------|----------|
| Claude Code | `~/.claude/consensus.json` | `plugins/consensus/consensus.config.json` | `~/.claude/.env` |
| Codex | `~/.codex/consensus.json` | `plugins/consensus/consensus.codex.config.json` | `~/.codex/.env` first, then `~/.claude/.env` read-only fallback |

Claude setup can smoke-test live model calls. Codex setup does not run live smoke tests unless you ask, because they consume credits.

Example Codex config:

```json
{
  "version": "1.7.0",
  "lead": {
    "id": "codex",
    "name": "Codex",
    "excluded_model_ids": ["gpt", "codex"]
  },
  "models": [
    {
      "id": "gemini",
      "name": "Gemini via Antigravity",
      "command": "agy",
      "resume_flag": "",
      "enabled": true
    },
    {
      "id": "kimi",
      "name": "Kimi K3.1 Pro",
      "command": "pi --provider openrouter --model moonshotai/kimi-k3.1-pro -p",
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
