# Security Policy

## Reporting a Vulnerability

If you discover a security vulnerability in this project, please report it responsibly.

**Do not open a public GitHub issue for security vulnerabilities.**

Instead, please email **support@altimate.ai** with:

- A description of the vulnerability
- Steps to reproduce
- Potential impact
- Suggested fix (if any)

We will acknowledge your report within 48 hours and work with you to understand and address the issue.

## Scope

This plugin is a set of markdown command definitions for Claude Code. Security concerns most relevant to this project include:

- **Prompt injection** — command definitions that could be manipulated to execute unintended actions
- **Credential exposure** — any path where API keys could be logged or leaked
- **Configuration tampering** — manipulation of `consensus.config.json` to execute arbitrary commands

## Best Practices for Users

- Store your `OPENROUTER_API_KEY` in `~/.claude/.env`, not in project files
- Do not commit `consensus.config.json` files that contain custom CLI commands with embedded credentials
- Review session artifacts in `/tmp/` and clean up after debugging sessions
