# Claude Code Skills

A collection of [Claude Code](https://claude.ai/code) skills for developer workflows.

## Available Skills

| Skill | Description |
|---|---|
| [pr-comprehension-check](pr-comprehension-check/) | Interactive multiple-choice quiz that tests whether you understand your AI-assisted code changes before creating a PR |

## Installation

### Via Plugin Marketplace (Recommended)

```bash
# In Claude Code, add this repo as a marketplace
/plugin marketplace add JacoboLansac/claude-skills

# Install a skill
/plugin install pr-comprehension-check@jacobo-claude-skills

# Reload to activate
/reload-plugins
```

### Manual Installation

```bash
# Clone and copy the skill you want
git clone https://github.com/JacoboLansac/claude-skills.git
cp -r claude-skills/pr-comprehension-check/skills/pr-comprehension-check ~/.claude/skills/
```

## Contributing

Feel free to open issues or PRs with suggestions, bug reports, or new skill ideas.

## License

MIT
