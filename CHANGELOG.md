# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

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
