# sloprails

Guard rails for that sloppin - A comprehensive claude.md configuration file for Claude Code projects.

## What is this?

This repository contains a battle-tested `claude.md` file that provides strict guardrails and best practices for working with Claude Code in software development projects. It enforces:

- Safe git practices (no auto-commits)
- Strict test coverage requirements
- Package management safety (venv-only, stable versions)
- DRY principles and code quality standards
- Minimal, surgical code changes

## Quick Start

### Download directly to your project

```bash
curl -o claude.md https://raw.githubusercontent.com/sadsfae/sloprails/main/claude.md
```

### Use with Claude Code

Place the downloaded `claude.md` file in the root of your project directory. Claude Code will automatically read and follow these rules when working in that directory.

## Bash Alias for Automatic Setup

Add this to your `~/.bashrc` to automatically download the latest claude.md before starting Claude Code:

```bash
alias claude='curl -s -o claude.md https://raw.githubusercontent.com/sadsfae/sloprails/main/claude.md 2>/dev/null && command claude'
```

After adding the alias:

1. Reload your bash configuration:

   ```bash
   source ~/.bashrc
   ```

2. Now when you run `claude` in any directory, it will:
   - Download the latest claude.md from this repository
   - Start Claude Code with those guardrails in place

## What's Inside

The `claude.md` file enforces:

### Git & Version Control

- No automatic commits or git operations
- All git actions must be explicitly performed by the user
- Changes are presented as diffs for manual review

### Package Management

- Mandatory virtual environment usage for all pip installs
- No packages newer than 10 days (stability requirement)
- Release date verification before installation

### Testing & Coverage

- Test coverage must never decrease
- Automatic coverage checks before/after changes
- New tests required for new functionality

### Code Quality

- No emojis or unicode characters
- Black formatting for Python
- Minimal comments (only when truly necessary)
- DRY principles strictly enforced

### Development Philosophy

- Simplicity first - minimum code to solve the problem
- Surgical changes - touch only what's necessary
- Think before coding - explicit assumptions, no silent decisions
- Goal-driven execution - define and verify success criteria

## Customization

Fork this repository and modify `claude.md` to fit your team's specific needs. Then update your bash alias to point to your fork's raw URL.

## License

See [LICENSE](LICENSE) file for details.

## Contributing

Improvements and suggestions are welcome. Please open an issue or pull request.
