# PHP 8.3 Codex image

Docker image for running Codex with a PHP-first toolchain.

Use this image for Laravel, Symfony, PHP packages, and PHP-first projects where Composer and PHP 8.3 must be available inside the Codex container.

---

## What is included

- PHP 8.3 CLI
- Composer 2
- Node.js 24
- npm
- Codex CLI
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

Node.js is installed only because Codex CLI runs through npm.

---

## Files

```text
php/
├── Dockerfile
├── docker-compose.yml
└── README.md
```

The compose file is designed to be run from the repository root:

```bash
docker compose -f php/docker-compose.yml ...
```

The compose build section should look like this:

```yaml
build:
  context: ..
  dockerfile: php/Dockerfile
```

`context: ..` means the repository root is the Docker build context.

`dockerfile: php/Dockerfile` points to this image’s Dockerfile from that root context.

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
mkdir -p coder/.codex coder/.composer coder/.cache/pip project_workspace
```

Build the PHP image:

```bash
docker compose -f php/docker-compose.yml build
```

Authenticate Codex:

```bash
docker compose -f php/docker-compose.yml run --rm codex codex login --device-auth
```

Start Codex:

```bash
docker compose -f php/docker-compose.yml run --rm codex
```

---

## Daily usage

Start Codex:

```bash
docker compose -f php/docker-compose.yml run --rm codex
```

Open a shell:

```bash
docker compose -f php/docker-compose.yml run --rm codex bash
```

Run one command:

```bash
docker compose -f php/docker-compose.yml run --rm codex php -v
```

Resume the last Codex session:

```bash
docker compose -f php/docker-compose.yml run --rm codex codex resume --last
```

---

## Verify the toolchain

Run:

```bash
docker compose -f php/docker-compose.yml run --rm codex php -v
docker compose -f php/docker-compose.yml run --rm codex composer --version
docker compose -f php/docker-compose.yml run --rm codex node -v
docker compose -f php/docker-compose.yml run --rm codex npm -v
docker compose -f php/docker-compose.yml run --rm codex python3 --version
docker compose -f php/docker-compose.yml run --rm codex codex --version
```

Expected PHP version:

```text
PHP 8.3.x
```

The PHP binary should come from the official PHP image path:

```bash
docker compose -f php/docker-compose.yml run --rm codex which php
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
  - ../coder/.codex:/home/node/.codex
  - ../coder/.composer:/home/node/.composer
  - ../coder/.cache/pip:/home/node/.cache/pip
```

This means Codex can edit only `../project_workspace`.

For real work, replace the workspace mount with the exact project path:

```yaml
volumes:
  - /Users/your-name/Work/my-laravel-project:/workspace
  - ../coder/.codex:/home/node/.codex
  - ../coder/.composer:/home/node/.composer
  - ../coder/.cache/pip:/home/node/.cache/pip
```

For stricter access, mount only a package or subdirectory:

```yaml
volumes:
  - /Users/your-name/Work/my-laravel-project/packages/MyPackage:/workspace
  - ../coder/.codex:/home/node/.codex
  - ../coder/.composer:/home/node/.composer
  - ../coder/.cache/pip:/home/node/.cache/pip
```

The mounted `/workspace` directory is the main directory Codex will work in.

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

For full application runtime, use the project’s real Docker/dev environment if it needs:

- MariaDB/MySQL
- PostgreSQL
- Redis
- queues
- mail services
- browser testing
- external APIs

Keep this image focused on Codex-assisted code work.

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

1. Docker container
2. explicit bind mounts
3. Codex approvals

Codex can access:

- the container filesystem
- `/workspace`
- explicitly mounted cache/config directories
- any other paths you add to `volumes`

Codex cannot access unmounted host paths.

Approvals do not expand Docker’s bind mount boundary.

---

## Recommended workflow

Before starting larger edits:

```bash
cd /path/to/your/project
git status
git add .
git commit -m "baseline before Codex"
```

Then run Codex with the project mounted as `/workspace`.

Inside Codex, prefer small, reviewable tasks:

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
docker compose -f php/docker-compose.yml build --no-cache
```

### Remove old containers

```bash
docker compose -f php/docker-compose.yml down --remove-orphans
```

### Check which image is used

```bash
docker compose -f php/docker-compose.yml config
```

Look for:

```yaml
build:
  context: ..
  dockerfile: php/Dockerfile
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
docker compose -f php/docker-compose.yml run --rm codex
```

Do not run a different compose file by accident.

If PHP unexpectedly shows 8.2, check that you are really using:

```yaml
dockerfile: php/Dockerfile
```

and not the root Node image.

---

## One-line summary

Use this image when Codex should work in a PHP 8.3 + Composer environment, while still being contained by Docker and limited to explicitly mounted directories.