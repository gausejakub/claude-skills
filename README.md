# Claude Skills

A collection of Claude Code plugins by [Jakub Gause](https://gause.dev).

## Available Skills

### laravel-worktrees

Set up git worktrees for Laravel + Herd with isolated databases, domains, and Vite ports. One command to create a fully isolated environment, one command to tear it down.

Each worktree gets:
- Its own database (MySQL, PostgreSQL, or SQLite)
- Its own Laravel Herd domain with HTTPS
- Its own Vite dev server port
- Claude Code hooks for automatic setup/cleanup

**Requirements:** Laravel project, Laravel Herd, a database (MySQL, PostgreSQL, or SQLite), git

## Installation

```bash
/plugin marketplace add gausejakub/claude-skills
/plugin install laravel-worktrees@gause-claude-skills
```

Then run `/setup-worktrees` in your Laravel project.

## Adding New Skills

Create a new directory under `plugins/`, add a `plugin.json` manifest and `skills/` directory, and add an entry to `.claude-plugin/marketplace.json`.
