# sloprails

Guard rails for that sloppin - agent skills and an agents/claude.md file for that hard sloppin.

## What is this?

### Claude.md or Agents.md

This repository contains a battle-tested `claude.md` file that provides strict guardrails and best practices for working with Claude Code in software development projects. It enforces:

- Safe git practices (no auto-commits)
- Strict test coverage requirements
- Package management safety (venv-only, stable versions)
- DRY principles and code quality standards
- Minimal, surgical code changes

### Agent skills

> [!NOTE]
> To properly handle threaded comments in Gerrithub you want to make an HTTP credential token and save it in `.gerrit_credentials` in your QUADS directory or home directory

* Add your `github` username to the skills and name in the skills
```bash
cd skills
sed -ie 's/YOURUSERNAME/YOUR_ACTUAL_GITHUB_USERNAME/g' */SKILL.md
sed -ie 's/Will Foster/Your Actual Display Name/g' */SKILL.md
```
* Copy the `skills` directory into `~/.agent/`
* Your agent harness should pick it up or look for it or you can mention it by path if needed

## Quick Start

### Download directly to your project

```bash
curl -o claude.md https://raw.githubusercontent.com/sadsfae/sloprails/main/claude.md
```

### Use with OMP

Use an omp `~/.bashrc` alias to set a system prompt that gets loaded on every `omp` launch.

```
  omp_dgx_glm53() {
      temp_agent="/tmp/omp_agents.md"
      curl -s -o "$temp_agent" https://raw.githubusercontent.com/sadsfae/sloprails/main/claude.md 2>/dev/null
      command omp --model remote-vllm-glm/dgx_hobo_default --append-system-prompt "$(cat "$temp_agent")" "$@"
  }
```

### Use with Claude Code

Place the downloaded `claude.md` file in the root of your project directory. Claude Code will automatically read and follow these rules when working in that directory.

## Bash Alias for Automatic Setup

Add this to your `~/.bashrc` to automatically download the latest claude.md before starting Claude Code:

```bash
alias claude='curl -s -o claude.md https://raw.githubusercontent.com/sadsfae/sloprails/main/claude.md 2>/dev/null && command claude " read claude.md in the current directory and then lets begin."'
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
