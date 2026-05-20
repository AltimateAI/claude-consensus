---
name: claude-consensus
description: Alias for Codex-led consensus plan review adapted from the Claude Consensus plugin. Use when the user invokes claude-consensus from Codex.
metadata:
  short-description: Claude Consensus alias for Codex
---

# Claude Consensus Alias

This skill aliases `consensus-plan-review` for users who know the original Claude plugin name.

When triggered, read `../consensus-plan-review/references/command.md` and follow it as the authoritative workflow.

Important:

- Codex is the lead reviewer.
- Do not call Codex/GPT as an external model.
- Use `~/.codex/consensus.json`.
