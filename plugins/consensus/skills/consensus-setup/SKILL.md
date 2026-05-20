---
name: consensus-setup
description: Inspect or refresh the Codex-led consensus configuration. Use when the user invokes consensus-setup or asks to configure claude-consensus for Codex.
metadata:
  short-description: Configure Codex consensus
---

# Consensus Setup

Follow the command body in `references/command.md`.

Important:

- Codex consensus config lives at `~/.codex/consensus.json`.
- It is intentionally separate from `~/.claude/consensus.json`.
- Codex/GPT must not be configured as an external panelist.
- Do not write `~/.claude/consensus.json` from Codex setup.
