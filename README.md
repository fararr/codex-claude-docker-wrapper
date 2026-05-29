# Codex in Docker on macOS

A practical setup for running **Codex only inside Docker** on macOS, with local auth storage and access limited to directories you explicitly mount.

The main idea is simple:

- Codex is not installed globally on macOS.
- Codex runs inside Docker.
- Codex can only access directories mounted into the container.
- Auth/config is stored locally in `./coder/.codex`.
- Different images can be used for different development workflows.

---

## Available images

This repository contains three Codex container variants.

### 1. Full Node-oriented image

Files:

```text
Dockerfile
docker-compose.yml
```

Use this as the default image for:

- JavaScript / TypeScript
- frontend work
- Node-based tooling
- mixed Node + PHP/Python helper work

It includes:

- Node.js 24
- Codex CLI
- PHP CLI and Composer
- Python 3
- useful CLI tools such as `ripgrep`, `jq`, `tree`, `shellcheck`, `sqlite3`

---

### 2. Minimal Node-oriented image

Files:

```text
Dockerfile.minimal
docker-compose.minimal.yml
```

Use this when you want a smaller image with only the basic tools needed to run Codex.

It includes:

- Node.js 22
- Codex CLI
- Git
- curl
- ripgrep
- basic shell/process tools

This image is useful when you do not need PHP, Composer, Python, or extra developer tooling.

---

### 3. PHP 8.3-oriented image

Files:

```text
php/Dockerfile
php/docker-compose.yml
php/README.md
```

Use this for:

- Laravel
- Symfony
- PHP packages
- PHP-first work with Composer
- mixed PHP + Python scripting

This image uses the official `php:8.3-cli-bookworm` base and then installs Node.js for the Codex CLI.

---

## Directory layout

Recommended structure:

```text
codex-docker/
├── Dockerfile
├── Dockerfile.minimal
├── docker-compose.yml
├── docker-compose.minimal.yml
├── README.md
├── .gitignore
├── coder/
│   ├── .codex/
│   │   ├── config.toml
│   │   └── auth.json
│   ├── .composer/
│   └── .cache/
│       └── pip/
├── php/
│   ├── Dockerfile
│   ├── docker-compose.yml
│   └── README.md
└── project_workspace/
```

### Directory meaning

- `coder/.codex/` stores Codex config, auth, and local state.
- `coder/.composer/` stores Composer cache for PHP-oriented containers.
- `coder/.cache/pip/` stores pip cache for Python tooling.
- `project_workspace/` is the default mounted workspace.
- `php/` contains the PHP 8.3-oriented image.

---

## Safety model

The real safety boundary is:

1. Docker container
2. Docker bind mounts
3. Codex approvals

That means:

- Codex can access the container filesystem.
- Codex can access only host directories you mount.
- Writable mounts can be edited, deleted, renamed, or git-modified.
- Approvals do not give Codex access to your whole Mac.
- “Outside the sandbox” means outside Codex’s inner sandbox, but still inside Docker.

### Practical conclusion

Docker bind mounts are the most important boundary.

Keep mounts narrow:

```yaml
volumes:
  - ./project_workspace:/workspace
```

Avoid mounting broad paths such as:

```yaml
volumes:
  - ~:/workspace
```

or:

```yaml
volumes:
  - /Users/your-name:/workspace
```

Prefer mounting only the exact repo or package Codex should access.

---

## First-time setup

From the repository root:

```bash
mkdir -p coder/.codex coder/.composer coder/.cache/pip project_workspace
```

Create `coder/.codex/config.toml`:

```toml
model = "gpt-5.4"
approval_policy = "on-request"
sandbox_mode = "workspace-write"
cli_auth_credentials_store = "file"

[projects."/workspace"]
trust_level = "trusted"
```

Recommended `.gitignore`:

```gitignore
coder/.codex/auth.json
coder/.codex/*.db
coder/.codex/logs/
coder/.composer/
coder/.cache/
```

---

## Build images

### Full Node image

```bash
docker compose -f docker-compose.yml build
```

### Minimal Node image

```bash
docker compose -f docker-compose.minimal.yml build
```

### PHP 8.3 image

```bash
docker compose -f php/docker-compose.yml build
```

---

## Authenticate Codex

Authenticate once. The auth is stored in `./coder/.codex`, so it can be reused by all image variants.

For the default image:

```bash
docker compose -f docker-compose.yml run --rm codex codex login --device-auth
```

For the PHP image:

```bash
docker compose -f php/docker-compose.yml run --rm codex codex login --device-auth
```

Device auth is usually the simplest login method from inside Docker.

---

## Start Codex

### Full Node image

```bash
docker compose -f docker-compose.yml run --rm codex
```

### Minimal Node image

```bash
docker compose -f docker-compose.minimal.yml run --rm codex
```

### PHP 8.3 image

```bash
docker compose -f php/docker-compose.yml run --rm codex
```

---

## Open a shell

### Full Node image

```bash
docker compose -f docker-compose.yml run --rm codex bash
```

### Minimal Node image

```bash
docker compose -f docker-compose.minimal.yml run --rm codex bash
```

### PHP 8.3 image

```bash
docker compose -f php/docker-compose.yml run --rm codex bash
```

Useful checks:

```bash
whoami
pwd
codex --version
node -v
ls -la /workspace
ls -la /home/node/.codex
```

For the full Node image:

```bash
php -v
composer --version
python3 --version
```

For the PHP image:

```bash
php -v
composer --version
node -v
npm -v
python3 --version
```

---

## Resume a Codex session

Resume the most recent session:

```bash
docker compose -f docker-compose.yml run --rm codex codex resume --last
```

With the PHP image:

```bash
docker compose -f php/docker-compose.yml run --rm codex codex resume --last
```

Resume a specific session:

```bash
docker compose -f docker-compose.yml run --rm codex codex resume YOUR_SESSION_ID
```

---

## Workspace mounts

The default root compose file mounts:

```yaml
volumes:
  - ./project_workspace:/workspace
  - ./coder/.codex:/home/node/.codex
```

That means Codex can edit `./project_workspace`.

You can replace this with a real project path:

```yaml
volumes:
  - /Users/your-name/Work/my-project:/workspace
  - ./coder/.codex:/home/node/.codex
```

For stricter work, mount only a package/subdirectory:

```yaml
volumes:
  - /Users/your-name/Work/my-project/packages/MyPackage:/workspace
  - ./coder/.codex:/home/node/.codex
```

### Read-only reference mounts

If Codex should only read a directory:

```yaml
volumes:
  - ./project_workspace:/workspace
  - ../reference_repo:/reference:ro
  - ./coder/.codex:/home/node/.codex
```

Use read-only mounts for documentation, examples, or reference repos.

---

## Recommended workflow

1. Mount only the exact directory Codex should access.
2. Make sure the project is in Git.
3. Commit a clean baseline before large Codex edits.
4. Keep `approval_policy = "on-request"`.
5. Review commands that install packages, delete files, modify Git state, or access unexpected paths.
6. Use the PHP 8.3 image for Laravel/Symfony/PHP-first work.
7. Use the default Node image for frontend/Node-first work.
8. Use the minimal image only when you want fewer tools inside the container.

---

## Known issues

### Docker context mismatch

If Docker still tries to connect to an old runtime such as OrbStack, switch Docker context:

```bash
docker context ls
docker context use desktop-linux
```

---

### Docker credential helper error

You may see:

```text
error getting credentials - err: exec: "docker-credential-desktop": executable file not found in $PATH
```

This is a Docker CLI configuration problem, not a Codex problem.

Usually it means `~/.docker/config.json` references a missing credential helper.

---

### Bubblewrap warning

Codex may print something like:

```text
Codex could not find bubblewrap on PATH...
Codex will use the bundled bubblewrap in the meantime.
```

This means Codex did not find a system-installed `bwrap` binary and will try its bundled fallback.

For this Docker setup, that is usually acceptable. The practical containment layer is Docker plus narrow bind mounts. Bubblewrap inside Docker Desktop can be unreliable because of namespace restrictions.

---

### “Outside the sandbox”

If Codex asks to run something “outside the sandbox”, that means outside Codex’s inner sandbox, but still inside the Docker container.

It does not mean Codex suddenly gets access to your whole Mac.

Approvals do not expand Docker’s bind mount boundary.

---

## Minimal compose note

Make sure `docker-compose.minimal.yml` points to the minimal Dockerfile:

```yaml
build:
  context: .
  dockerfile: Dockerfile.minimal
```

Also prefer unique image/container names for each variant to avoid collisions, for example:

```yaml
image: codex-local:minimal-0.135.0
container_name: codex-minimal
```

---

## One-line summary

Codex runs only inside Docker, reuses file-based auth from `./coder/.codex`, edits only explicitly mounted directories, and relies on Docker isolation plus approval prompts rather than broad host access.