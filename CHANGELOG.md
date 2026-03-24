# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

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
