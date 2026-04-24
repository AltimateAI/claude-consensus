# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.4.0] - 2026-04-24

### Added

- **DeepSeek V4 Pro** as 9th model (`openrouter/deepseek/deepseek-v4-pro`)

### Changed

- Upgrade to latest model versions across the panel:
  - Kimi K2.5 -> K2.6 (`openrouter/moonshotai/kimi-k2.6`)
  - Grok 4 -> Grok 4.20 (`openrouter/x-ai/grok-4.20-beta` — Kilo maps OpenRouter's `x-ai/grok-4.20` under the `-beta` suffix in its model registry)
  - Qwen 3.5 Plus -> Qwen 3.6 Plus (`openrouter/qwen/qwen3.6-plus`)
  - GLM-5 -> GLM-5.1 (`zai-coding-plan/glm-5.1`)
- Default minimum quorum raised to 10 (100% — all 9 models + Claude must respond)

### Removed

- Nemotron 120B model — low contribution on unique findings, dropped from default panel

### Migration

Users with existing `~/.claude/consensus.json` should re-run `/consensus-setup` or manually remove
the `nemotron` entry. Qwen users on the broken `qwen3.6:free` path should update to
`openrouter/qwen/qwen3.6-plus` (paid tier; no free `qwen3.6` variant is available).

## [1.2.0] - 2026-03-30

### Added

- Generic `/consensus:review` command for document and general topic reviews
- Multi-directory support (`--dirs`) for cross-repo reviews and plans across all three commands
- CodeRabbit static analysis as supplementary reviewer in code-review (runs `coderabbit review --plain`)
- Per-CLI directory flags: `--add-dir` for Codex, `--include-directories` for Gemini/Qwen

### Changed

- Plugin description updated to reflect three review commands

## [1.1.0] - 2026-03-24

### Added

- Nemotron 120B and MiMo V2 Pro models (9 external models total)
- Compaction resilience for code-review and plan-review commands
- Qwen native CLI detection in setup wizard
- Unique progress files per session (no collisions on back-to-back runs)
- Status-aware resume (only resumes IN_PROGRESS, ignores COMPLETED/FAILED)

### Changed

- MiniMax bumped from M2.5 to M2.7
- Qwen now has both OpenRouter and native CLI paths (was native-only)
- Setup wizard updated to 9 models with qwen CLI detection
- `stat -f%z` replaced with cross-platform `wc -c` for Linux compatibility
- Default quorum raised from 5 to 6
- Removed token budget constraints from compaction resilience

### Fixed

- Setup wizard rule 3 now correctly lists codex, gemini, and qwen for native CLI support

## [1.0.0] - 2026-03-05

### Added

- Multi-model consensus code review via `/code-review`
- Multi-model consensus plan review via `/plan-review`
- Interactive setup wizard via `/consensus-setup`
- Support for 7 external models (GPT 5.4 Codex, Gemini 3.1 Pro, Kimi K2.5, Grok 4, MiniMax M2.5, GLM-5, Qwen 3.5 Plus)
- OpenRouter integration via Kilo CLI for single-key access to all models
- Configurable quorum with graceful degradation
- Independent review phases with no cross-contamination
- Structured synthesis and convergence rounds
- Session artifacts saved to `/tmp/` for debugging
