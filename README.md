# Claude Skills

A collection of Claude Code skills to enhance your development workflow.

## Marketplace Overview

This repository contains professionally-crafted Claude Code skills that automate common development tasks. Each skill is designed to integrate seamlessly with Claude Code and can be installed via the marketplace or manually.

## Available Skills

### 📋 [Guideline Refresher](skills/guideline-refresher/)

Auto-refresh coding guidelines based on codebase patterns, PR reviews, and approved code.

**What it does:** Analyzes your git history, merged PRs, and review comments to generate evidence-based coding guidelines that reflect your actual practices—not outdated documentation.

**Perfect for:**
- Keeping guidelines current after migrations or refactorings
- Documenting actual team patterns from approved code
- Generating area-specific rules (frontend, backend, utils, etc.)

**Links:**
- 📖 [Full Documentation](skills/guideline-refresher/README.md)
- 🚀 [Usage Examples](EXAMPLES.md)

## Installation

### From Marketplace (Recommended)

```bash
# Add marketplace
/plugin marketplace add jjholmes927/jjholmes927-claude-skills

# Install plugin
/plugin install

# Select: jjholmes927-claude-skills
# Choose skills to install
```

### Manual Installation

```bash
# Clone and install all skills
git clone https://github.com/jjholmes927/jjholmes927-claude-skills.git
cd jjholmes927-claude-skills
./install.sh
```

## Quick Start

Once installed, skills work automatically:

```
You: "Refresh guidelines for frontend/components"
Claude: [Uses guideline-refresher skill automatically]
```

Or run directly from command line:
```bash
~/.claude/skills/guideline-refresher/refresh.sh --area frontend/components
```

## Contributing

Contributions welcome! To add a new skill:

1. Create a directory under `skills/{skill-name}/`
2. Add `SKILL.md` with skill definition
3. Add implementation files (Python, shell scripts, etc.)
4. Add `README.md` with documentation
5. Update this README with the new skill
6. Update `plugin.json` with the skill metadata

## License

MIT - See LICENSE file for details
