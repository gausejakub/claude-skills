---
name: setup-worktrees
description: Set up git worktrees for Laravel + Herd with isolated databases, domains, and Vite ports. Use when the user wants to configure worktree-based development environments for their Laravel project.
disable-model-invocation: true
---

# Set Up Git Worktrees for Laravel + Herd

You are setting up a git worktree system for a Laravel project served by Laravel Herd. Each worktree will get its own database, Herd domain with HTTPS, and Vite dev server port.

## Before You Start

1. Confirm you are in the root of a Laravel project (check for `artisan`, `composer.json`)
2. Confirm `.env` exists and has `APP_URL`, `DB_DATABASE`
3. Detect the database driver from `DB_CONNECTION` in `.env` (default: `mysql`). Supported: `mysql`, `pgsql`, `sqlite`
4. For `mysql`/`pgsql`: confirm the `.env` has `DB_USERNAME`, `DB_HOST`, `DB_PORT` and that the database server is reachable
5. Confirm Herd is available (`herd --version`)
6. Detect the JS package manager from lockfile (`pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, `package-lock.json` → npm)

If anything is missing, tell the user what they need and stop.

## What to Create

Create all scripts in `.claude/scripts/`. Create the directory if it doesn't exist.

Add `.claude/worktrees/` to `.gitignore` if not already there.

### Script 1: `setup-worktree.sh`

Full environment setup for a worktree. Takes a worktree path as argument. Must be idempotent (skip if `.worktree-ready` marker exists).

Steps:
1. Read `APP_URL` from the **main** worktree's `.env` to derive the base domain and app name
2. Derive subdomain from the worktree folder name: `{folder}.{base-app-name}` → `{folder}.{base-app-name}.test`
3. Copy `.env` from the main project if it doesn't exist in the worktree
4. Set `APP_URL` to `https://{subdomain}.test`
5. Set `APP_HOSTNAME` to `{subdomain}.test` (if the key exists in `.env`)
6. Derive database name: `{base-app-name}_{folder_with_underscores}` and set `DB_DATABASE`
7. Find a free Vite port in range 5100–5199 by checking other worktree `.env` files AND active listeners (`lsof`). Set `VITE_PORT`
8. Install PHP dependencies: `composer install --no-interaction --quiet`
9. Install JS dependencies using the detected package manager
10. Generate application key: `php artisan key:generate --no-interaction --quiet`
11. Create the worktree database based on `DB_CONNECTION`:
    - **mysql**: `mysql -u $DB_USERNAME -p$DB_PASSWORD -h $DB_HOST -P $DB_PORT -e "CREATE DATABASE IF NOT EXISTS \`$DB_DATABASE\`"`
    - **pgsql**: `PGPASSWORD=$DB_PASSWORD createdb -U $DB_USERNAME -h $DB_HOST -p $DB_PORT $DB_DATABASE` (ignore error if it already exists)
    - **sqlite**: set `DB_DATABASE` to the worktree's `database/database.sqlite` absolute path, then `touch` the file
12. Run migrations: `php artisan migrate --seed --no-interaction --force`
13. If Passport is installed, run `php artisan passport:install --no-interaction` (don't fail if not installed)
14. Create storage link: `php artisan storage:link --no-interaction --force`
15. Set permissions on `storage/` and `bootstrap/cache/` (775)
16. Patch the `package.json` dev script to include `--port {VITE_PORT}` (use node to read/write JSON)
17. Link to Herd: `herd link {link-name}` and `herd secure {link-name}`
18. After Herd link/secure, re-set `APP_URL` and `APP_HOSTNAME` (Herd sometimes overwrites `.env`)
19. Touch `.worktree-ready` marker file
20. Print summary: branch, path, URL, database, Vite port, dev server command

### Script 2: `claude-worktree.sh`

User-facing create command. Usage: `./claude-worktree.sh [branch-name]`

- If no branch name, generate one: `wt-YYYYMMDD-HHMMSS`
- Sanitize for folder name (replace `/` with `-`, lowercase)
- If worktree exists and is set up, just cd and start Claude
- If worktree exists but not set up, show error
- Create git worktree (use existing branch if it exists, otherwise create new)
- Run `setup-worktree.sh`
- Start Claude Code in the worktree: `cd "$WORKTREE_DIR" && exec claude`

### Script 3: `cleanup-worktree.sh`

Standalone cleanup. Takes a worktree path as argument. Reads the worktree's `.env` to determine what to clean up.

- Drop the worktree database based on `DB_CONNECTION` from the worktree's `.env`:
  - **mysql**: `mysql -u $DB_USERNAME -p$DB_PASSWORD -h $DB_HOST -P $DB_PORT -e "DROP DATABASE IF EXISTS \`$DB_DATABASE\`"`
  - **pgsql**: `PGPASSWORD=$DB_PASSWORD dropdb -U $DB_USERNAME -h $DB_HOST -p $DB_PORT --if-exists $DB_DATABASE`
  - **sqlite**: `rm -f $DB_DATABASE`
- Unsecure Herd: `herd unsecure {link-name}`
- Unlink Herd: `herd unlink {link-name}`

### Script 4: `claude-worktree-remove.sh`

User-facing remove command. Usage: `./claude-worktree-remove.sh [name]`

- If no argument, list available worktrees and exit
- Run `cleanup-worktree.sh` on the worktree
- Remove git worktree: `git worktree remove --force`
- Delete the branch: `git branch -D`

### Script 5: `ensure-worktree-setup.sh`

Fast pre-check for Claude Code hooks. Two checks only:
- If `.git` is not a file (i.e. we're in the main repo, not a worktree), exit immediately
- If `.worktree-ready` exists, exit immediately
- Otherwise run `setup-worktree.sh` on current directory

### Script 6: `detect-worktree-remove.sh`

Claude Code hook for auto-cleanup. Receives `$CLAUDE_TOOL_INPUT` JSON.
- Extract `command` field using python3
- Check if it matches `git worktree remove`
- Extract the worktree path (handle `--force` flag)
- Run `cleanup-worktree.sh` on the path

## Claude Code Hooks

Update `.claude/settings.json` to add (or merge into existing) `PreToolUse` hooks on the `Bash` matcher:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "MAIN=$(git worktree list 2>/dev/null | head -1 | awk '{print $1}') && [ -n \"$MAIN\" ] && \"$MAIN/.claude/scripts/ensure-worktree-setup.sh\" || true"
          },
          {
            "type": "command",
            "command": "MAIN=$(git worktree list 2>/dev/null | head -1 | awk '{print $1}') && [ -n \"$MAIN\" ] && \"$MAIN/.claude/scripts/detect-worktree-remove.sh\" \"$CLAUDE_TOOL_INPUT\" || true"
          }
        ]
      }
    ]
  }
}
```

If `.claude/settings.json` already exists with other hooks, merge carefully — don't overwrite existing hooks.

## Laravel Side: Vite Port

Check the Blade layout file that loads Vite assets. If there's a hardcoded port `5173` in a dev-only script tag or custom directive, replace it with:

```php
$port = Env::get('VITE_PORT', '5173');
```

This ONLY affects the development code path. If the project uses Laravel's built-in `@vite()` directive, skip this step.

## Important Rules

- All scripts must use `set -euo pipefail`
- All scripts must be `chmod +x`
- Never modify the main project's `.env` or database
- Database credentials must come from the worktree's `.env`, never hardcoded
- Use `--no-interaction` on all Artisan commands
- The setup script must be idempotent
- After creating all scripts, show the user how to use them
