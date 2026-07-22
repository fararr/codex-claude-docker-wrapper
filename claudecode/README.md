# PHP 8.3 Claude Code image

Docker image for running Claude Code with a PHP-first toolchain.

Use this image for Laravel, Symfony, PHP packages, and PHP-first projects where Composer and PHP 8.3 must be available inside the Claude Code container.

---

## What is included

- PHP 8.3 CLI
- Composer 2
- Node.js 24
- npm
- Claude Code CLI
- Python 3
- common CLI tools:
  - `git`
  - `curl`
  - `ripgrep`
  - `fd-find`
  - `jq`
  - `tree`
  - `sqlite3`
  - `shellcheck`
  - `zip`
  - `unzip`
  - `make`
  - `patch`

The image is based on:

```dockerfile
FROM php:8.3-cli-bookworm
```

Node.js is installed only to install the Claude Code npm package. The `claude`
binary itself is native and does not use Node at runtime.

---

## Files

```text
claudecode/
├── Dockerfile
├── docker-compose.yml
└── README.md
```

The compose file is designed to be run from the repository root:

```bash
docker compose -f claudecode/docker-compose.yml ...
```

The compose build section should look like this:

```yaml
build:
  context: ..
  dockerfile: claudecode/Dockerfile
```

`context: ..` means the repository root is the Docker build context.

`dockerfile: claudecode/Dockerfile` points to this image's Dockerfile from that root context.

---

## Best fit

Use this image when the workspace is mainly:

- Laravel
- Symfony
- Composer package development
- generic PHP CLI work
- PHP application maintenance
- mixed PHP + Python scripting

For Node-first/frontend work, use the root Node image instead.

---

## First-time setup

From the repository root:

```bash
mkdir -p coder/.claude coder/.composer coder/.cache/pip project_workspace
```

Build the PHP image:

```bash
docker compose -f claudecode/docker-compose.yml build
```

Authenticate Claude Code (one time). Run `claude` and follow the login prompts;
the token is written to `coder/.claude/.credentials.json` and reused on every
subsequent run — the same persisted-token model you used for Codex's
`coder/.codex/auth.json`:

```bash
docker compose -f claudecode/docker-compose.yml run --rm claude-code claude
```

For a long-lived, non-interactive token (Pro/Max), you can instead run:

```bash
docker compose -f claudecode/docker-compose.yml run --rm claude-code claude setup-token
```

Start Claude Code:

```bash
docker compose -f claudecode/docker-compose.yml run --rm claude-code
```

---

## Daily usage

Start Claude Code:

```bash
docker compose -f claudecode/docker-compose.yml run --rm claude-code
```

Open a shell:

```bash
docker compose -f claudecode/docker-compose.yml run --rm claude-code bash
```

Run one command:

```bash
docker compose -f claudecode/docker-compose.yml run --rm claude-code php -v
```

Continue the most recent Claude Code session in the workspace:

```bash
docker compose -f claudecode/docker-compose.yml run --rm claude-code claude --continue
```

Pick an earlier session to resume:

```bash
docker compose -f claudecode/docker-compose.yml run --rm claude-code claude --resume
```

Run headless (non-interactive) inside the sandbox. Because the container *is*
the sandbox and runs as uid 1000 (not root), skipping approval prompts is
reasonable here:

```bash
docker compose -f claudecode/docker-compose.yml run --rm claude-code \
  claude --dangerously-skip-permissions -p "run the test suite and fix failures"
```

---

## Verify the toolchain

Run:

```bash
docker compose -f claudecode/docker-compose.yml run --rm claude-code php -v
docker compose -f claudecode/docker-compose.yml run --rm claude-code composer --version
docker compose -f claudecode/docker-compose.yml run --rm claude-code node -v
docker compose -f claudecode/docker-compose.yml run --rm claude-code npm -v
docker compose -f claudecode/docker-compose.yml run --rm claude-code python3 --version
docker compose -f claudecode/docker-compose.yml run --rm claude-code claude --version
```

Expected PHP version:

```text
PHP 8.3.x
```

The PHP binary should come from the official PHP image path:

```bash
docker compose -f claudecode/docker-compose.yml run --rm claude-code which php
```

Expected:

```text
/usr/local/bin/php
```

---

## Workspace mount

By default, the PHP compose file can mount:

```yaml
volumes:
  - ../project_workspace:/workspace
  - ../coder/.claude:/home/node/.claude
  - ../coder/.composer:/home/node/.composer
  - ../coder/.cache/pip:/home/node/.cache/pip
```

This means Claude Code can edit only `../project_workspace`.

For real work, replace the workspace mount with the exact project path:

```yaml
volumes:
  - /Users/your-name/Work/my-laravel-project:/workspace
  - ../coder/.claude:/home/node/.claude
  - ../coder/.composer:/home/node/.composer
  - ../coder/.cache/pip:/home/node/.cache/pip
```

For stricter access, mount only a package or subdirectory:

```yaml
volumes:
  - /Users/your-name/Work/my-laravel-project/packages/MyPackage:/workspace
  - ../coder/.claude:/home/node/.claude
  - ../coder/.composer:/home/node/.composer
  - ../coder/.cache/pip:/home/node/.cache/pip
```

The mounted `/workspace` directory is the main directory Claude Code will work in.

---

## Laravel / Symfony notes

This image is intended for source-code work, not for running a full production stack.

Good uses:

- inspect and edit Laravel/Symfony code
- run Composer commands
- run PHP scripts
- run tests that do not require external services
- run static checks or formatters
- inspect SQLite-based local fixtures

For full application runtime, use the project's real Docker/dev environment if it needs:

- MariaDB/MySQL
- PostgreSQL
- Redis
- queues
- mail services
- browser testing
- external APIs

Keep this image focused on Claude-Code-assisted code work.

---

## Auth / token

Claude Code stores its credentials inside the mounted config dir:

```text
coder/.claude/.credentials.json
```

Inside the container:

```text
/home/node/.claude/.credentials.json
```

This is the direct analog of Codex's `coder/.codex/auth.json`. Log in once (see
First-time setup) and the token persists across container rebuilds and runs.
Treat this file like a secret: keep `coder/.claude/` out of version control.

The image sets `CLAUDE_CONFIG_DIR=/home/node/.claude` so credentials, settings,
`.claude.json`, session history, and todos all live under this single mount.
Auto-updates are disabled (`DISABLE_AUTOUPDATER=1`) so the pinned CLI version
stays put.

---

## Composer cache

Composer cache is mounted from:

```text
../coder/.composer
```

Inside the container:

```text
/home/node/.composer
```

This speeds up repeated Composer operations across container rebuilds and runs.

---

## Python cache

pip cache is mounted from:

```text
../coder/.cache/pip
```

Inside the container:

```text
/home/node/.cache/pip
```

This is useful for small Python helper scripts or tooling.

---

## Safety model

This image follows the same safety model as the root images:

1. Docker container (`cap_drop: ALL`, `no-new-privileges`, non-root uid 1000)
2. explicit bind mounts
3. Claude Code approvals

Claude Code can access:

- the container filesystem
- `/workspace`
- explicitly mounted cache/config directories
- any other paths you add to `volumes`

Claude Code cannot access unmounted host paths.

Approvals do not expand Docker's bind mount boundary. `--dangerously-skip-permissions`
only removes the approval prompts inside the container; it does not widen the
mount boundary, which is why it is acceptable to use here but not on the host.

---

## Recommended workflow

Before starting larger edits:

```bash
cd /path/to/your/project
git status
git add .
git commit -m "baseline before Claude Code"
```

Then run Claude Code with the project mounted as `/workspace`.

Inside Claude Code, prefer small, reviewable tasks:

- one bugfix
- one refactor
- one feature slice
- one test improvement
- one documentation update

Review generated diffs before committing.

---

## Build troubleshooting

### Rebuild without cache

```bash
docker compose -f claudecode/docker-compose.yml build --no-cache
```

### Remove old containers

```bash
docker compose -f claudecode/docker-compose.yml down --remove-orphans
```

### Check which image is used

```bash
docker compose -f claudecode/docker-compose.yml config
```

Look for:

```yaml
build:
  context: ..
  dockerfile: claudecode/Dockerfile
```

### Test the base image directly

```bash
docker run --rm php:8.3-cli-bookworm php -v
```

This should print PHP 8.3.x.

---

## Important path note

Run this compose file from the repository root:

```bash
docker compose -f claudecode/docker-compose.yml run --rm claude-code
```

Do not run a different compose file by accident.

If PHP unexpectedly shows 8.2, check that you are really using:

```yaml
dockerfile: claudecode/Dockerfile
```

and not the root Node image.

---

## One-line summary

Use this image when Claude Code should work in a PHP 8.3 + Composer environment, while still being contained by Docker and limited to explicitly mounted directories.