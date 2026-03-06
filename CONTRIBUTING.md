# Contributing to claude-consensus

Thanks for your interest in contributing! This guide will help you get started.

## Ways to Contribute

- **Report bugs** via [GitHub Issues](https://github.com/AltimateAI/claude-consensus/issues)
- **Suggest features** via [GitHub Issues](https://github.com/AltimateAI/claude-consensus/issues)
- **Submit pull requests** for bug fixes, new models, or improvements
- **Improve documentation** — typo fixes, clarifications, examples

## Getting Started

1. Fork the repository
2. Clone your fork:
   ```bash
   git clone https://github.com/YOUR_USERNAME/claude-consensus.git
   cd claude-consensus
   ```
3. Create a branch:
   ```bash
   git checkout -b feat/your-feature-name
   ```
4. Install the plugin locally in Claude Code:
   ```
   /plugin marketplace add /path/to/claude-consensus
   /plugin install consensus
   ```

## Project Structure

```
.claude-plugin/marketplace.json        # Plugin marketplace registration
plugins/consensus/
  .claude-plugin/plugin.json           # Plugin metadata
  AGENTS.md                            # Agent behavior documentation
  consensus.config.json                # Default model configuration
  commands/
    code-review.md                     # /code-review slash command
    consensus-setup.md                 # /consensus-setup slash command
    plan-review.md                     # /plan-review slash command
```

## Making Changes

### Adding a New Model

1. Add the model entry to `plugins/consensus/consensus.config.json`
2. Update the supported models table in `README.md`
3. Test that the model responds correctly via `/consensus-setup` smoke test

### Modifying Commands

Command files in `plugins/consensus/commands/` are markdown files that Claude Code interprets as slash command definitions. When editing:

- Preserve the structured format (phases, templates, rules)
- Test the full workflow end-to-end after changes
- Ensure quorum logic still works with your changes

### Updating Configuration

- Changes to `consensus.config.json` affect default settings for all new users
- Ensure backwards compatibility — don't remove fields existing configs depend on

## Pull Request Process

1. **Keep PRs focused** — one feature or fix per PR
2. **Write a clear description** explaining what changed and why
3. **Test your changes** — run the relevant slash commands and verify the workflow
4. **Update documentation** if your change affects usage or configuration
5. **Follow existing patterns** — match the style and structure of existing files

### Commit Messages

Use [conventional commits](https://www.conventionalcommits.org/):

```
feat: add support for new model X
fix: handle timeout when model is unavailable
docs: clarify quorum configuration
chore: update .gitignore
```

## Testing

Since this is a markdown-based plugin, testing is manual:

1. Install the plugin locally from your branch
2. Run `/consensus-setup` and verify configuration works
3. Run `/code-review` on a sample diff
4. Run `/plan-review` on a sample task
5. Verify quorum enforcement with reduced model count

## Code of Conduct

This project follows the [Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md). By participating, you agree to uphold this standard.

## Questions?

Open an issue or reach out at support@altimate.ai.
