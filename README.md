# claude-consensus

**Multi-model code review and plan review for Claude Code**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Plugin-blueviolet)](https://claude.ai/claude-code)

Multiple AI models independently review code or plan implementations, then converge on consensus through structured synthesis and approval rounds. Configurable — works with as few as Claude + 1 external model.

## Quick Start

Just tell Claude:

> "Set up claude-consensus: https://github.com/AltimateAI/claude-consensus"

That's it for most users. The rest of this README is for understanding what's happening under the hood.

### Manual Setup

```bash
# 1. Install the plugin
/plugin marketplace add AltimateAI/claude-consensus
/plugin install consensus

# 2. Configure your models
/consensus-setup

# 3. Use it
/code-review "staged changes"
/plan-review "Add caching to the API layer"
```

## Prerequisites

| Requirement | Required? | Notes |
|-------------|-----------|-------|
| [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI | **Yes** | The host environment |
| [Kilo CLI](https://github.com/kilo-org/kilo) | Recommended | Routes all 7 models via OpenRouter with 1 API key |
| [OpenRouter](https://openrouter.ai) API key | Recommended | Required if using Kilo CLI |
| Native CLIs (`codex`, `gemini`) | Optional | Alternative to OpenRouter for specific models |

**Minimal setup**: Claude + 1 external model is enough for consensus reviews.

## Supported Models

| Model | Provider | OpenRouter ID | Native CLI |
|-------|----------|---------------|------------|
| Claude | Anthropic | (built-in) | Claude Code |
| GPT 5.2 Codex | OpenAI | `openai/gpt-5.2-codex` | `codex` |
| Gemini 3.1 Pro | Google | `google/gemini-3.1-pro-preview` | `gemini` |
| Kimi K2.5 | Moonshot | `moonshotai/kimi-k2.5` | — |
| Grok 4 | xAI | `x-ai/grok-4` | — |
| MiniMax M2.5 | MiniMax | `minimax/minimax-m2.5` | — |
| GLM-5 | Zhipu AI | `z-ai/glm-5` | — |
| Qwen 3.5 Plus | Alibaba | `qwen/qwen3.5-plus-02-15` | — |

## Installation

### From GitHub
```bash
# Via CLI
claude plugin marketplace add AltimateAI/claude-consensus
claude plugin install consensus

# Or in-session
/plugin marketplace add AltimateAI/claude-consensus
/plugin install consensus
```

### From Source (local clone)
```bash
git clone https://github.com/AltimateAI/claude-consensus.git
```

Then in a Claude Code session:
```
/plugin marketplace add /path/to/claude-consensus
/plugin install consensus
```

## Configuration

### Setup Wizard (Recommended)

Run `/consensus-setup` to interactively configure:
- **Provider**: OpenRouter (1 key for all models), native CLIs, or both
- **API key**: Saved to `~/.claude/.env`
- **Models**: Choose which of the 7 external models to enable
- **Quorum**: Minimum models that must respond for a valid review
- **Smoke test**: Verifies each model responds before finalizing

### Config Location

| Path | Purpose |
|------|---------|
| `~/.claude/consensus.json` | User config (created by `/consensus-setup`) |
| `plugins/consensus/consensus.config.json` | Plugin defaults (fallback) |

### Config Format

```json
{
  "version": "1.0.0",
  "models": [
    {
      "id": "gpt",
      "name": "GPT 5.2 Codex",
      "command": "kilo run -m openrouter/openai/gpt-5.2-codex --auto",
      "resume_flag": "-c",
      "enabled": true
    }
  ],
  "min_quorum": 5
}
```

### Manual Configuration

Edit `~/.claude/consensus.json` directly to:
- Enable/disable models (`"enabled": true/false`)
- Change CLI commands
- Adjust quorum

Or re-run `/consensus-setup` anytime to reconfigure.

## Commands

| Command | Purpose |
|---------|---------|
| `/consensus-setup` | Configure models, API keys, and quorum |
| `/code-review [target]` | Multi-model code review with consensus |
| `/plan-review [task]` | Multi-model implementation planning with consensus |

## Usage

```
# Code review
/code-review "staged changes"
/code-review PR #123
/code-review "last 3 commits"
/code-review app/service/foo.py

# Plan review
/plan-review "Add caching to the API layer"
/plan-review "Refactor auth to use JWT"
```

## How It Works

```
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│ Claude  │  │ Model 1 │  │ Model 2 │  │ Model N │
└────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘
     │            │            │            │
     ▼            ▼            ▼            ▼
  Phase 1: Independent Review/Planning (parallel)
     └────────────┴────────────┴────────────┘
                       │
                       ▼
            Phase 2: Synthesis
         (consensus, conflicts, comparison table)
                       │
                       ▼
            Phase 3: Convergence
         (APPROVE / CHANGES NEEDED, max 2 rounds)
                       │
                       ▼
              Final Result with Attribution
```

- **Quorum**: Configurable (default: 5). Strict majority of participants must respond.
- **Graceful degradation**: Unavailable models are skipped at runtime if quorum is still met.
- **Session artifacts**: Saved in `/tmp/` for debugging.
- **Independent reviews**: Each model reviews with no cross-contamination.

## Contributing

Contributions are welcome! Please read the [Contributing Guide](CONTRIBUTING.md) before submitting a PR.

This project follows the [Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md).

## License

MIT License. Copyright (c) 2026 [Altimate AI](https://altimate.ai/).
