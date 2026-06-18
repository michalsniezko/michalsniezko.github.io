---
layout: default
title: Docker Compose Override for Local Development
parent: Infrastructure & CI/CD
nav_order: 13
---

## Docker Compose Override for Local Development

A Docker image baked for CI and production is not ideal for local development. It has stale code, files owned by the wrong user, and no live reload. But you also cannot just add bind mounts and port mappings to the committed `compose.yaml` - that would break CI.

The solution is a two-layer Compose setup: a committed `compose.yaml` that works identically everywhere, and a gitignored `compose.override.yaml` that adds the local conveniences each developer needs on their own machine.

---

## How Docker Compose Auto-Merging Works

When you run `docker compose up`, Docker Compose automatically looks for and merges two files in order:

1. `compose.yaml` (or `docker-compose.yaml`)
2. `compose.override.yaml` (if it exists)

The override file is a partial Compose definition - it only needs to contain the keys you want to change. Compose deep-merges it on top of the base file. Keys in the override win; everything else stays as defined in the base.

This means the override is invisible to anything that doesn't have the file. CI runners don't have it - they see only `compose.yaml`.

---

## The Problem the Override Solves

### Stale code after every edit

The production Dockerfile bakes source code into the image at build time:

```dockerfile
FROM base AS app_with_tests
USER www-data
COPY --chown=www-data:www-data . /app
RUN composer install --no-progress --prefer-dist --no-interaction
```

In CI this is correct - the image is the artifact. Locally it means every file change requires a full `docker compose build` before it takes effect.

A bind mount fixes this by replacing the baked `/app` directory with your live working copy:

```yaml
services:
  php-fpm-test:
    volumes:
      - ./:/app   # host working copy mounts over the baked /app
```

Now edits show up immediately inside the container without a rebuild.

### File ownership mismatch

The Dockerfile bakes files as `www-data` (uid 33). But the container runs as your host user:

```yaml
# compose.yaml
services:
  php-fpm-test:
    user: ${USERID:-1000}:${GROUPID:-1000}
```

Without a bind mount, the container starts as uid 1000 but `/app` is owned by uid 33. The container cannot write to `var/cache`, `tests/_output`, or any generated file - exactly the "Path for output is not writable" errors you see locally.

The bind mount replaces the baked `/app` with your host working copy, which your host user owns. The ownership mismatch disappears.

### No port access to the dev server

Locally you need to reach the web server from your browser. In CI you do not. The override adds the port mapping:

```yaml
services:
  apache:
    volumes:
      - ./:/app
    ports:
      - "1080:80"
```

---

## The .dist Template

`compose.override.yaml` is gitignored so developers can customise it without dirtying the repo. But new developers need a starting point. The solution is to commit a template:

```
compose.yaml              # committed - the base, works in CI and prod
compose.override.dist.yaml  # committed - the template developers copy from
compose.override.yaml     # gitignored - the actual local file, never committed
```

The `.gitignore` entry:

```
compose.override.yaml
```

The `compose.override.dist.yaml` content - the minimal override that makes local development work:

```yaml
services:
  php-fpm-test:
    volumes:
      - ./:/app
  apache:
    volumes:
      - ./:/app
    ports:
      - "1080:80"
```

First-time setup for a new developer is a single copy:

```bash
cp compose.override.dist.yaml compose.override.yaml
```

After that, `make up` and `make test` work correctly.

---

## Why CI Must Not Use the Override

CI has no host working copy with `vendor/`. Dependencies are baked into the image during the Docker build step, not installed on the CI runner. If CI somehow loaded the override:

- The bind mount `./:/app` would replace the baked `/app` (which has `vendor/`) with the CI workspace (which does not)
- The application would fail to start with "Class not found" errors
- The `codecept` service runs as `www-data` (uid 33) in CI - which has write access to the baked `/app` - and would break if the bind mount replaced it with files owned by a different uid

CI is safe because the file simply does not exist on the runner. There is no flag to set, no profile to disable - the absence of the file is the mechanism.

---

## Compose Profiles

A related pattern for managing multiple environments is Docker Compose profiles. Rather than separate files, profiles let you tag services and select which ones to start:

```yaml
services:
  php-fpm-test:
    profiles:
      - dev
      - ci

  apache:
    profiles:
      - dev
      - ci
      - prod

  quality:
    extends:
      service: php-fpm-test
    command: sh -c 'bin/phpcs && bin/phpstan analyse'
    profiles:
      - tools

  codecept:
    extends:
      service: php-fpm-test
    user: www-data   # CI: baked /app is www-data-owned, no host mount
    command: sh -c 'bin/codecept run'
    profiles:
      - tools
```

Profiles and the override file work independently. Profiles select which services run; the override file adjusts how services are configured. In this setup:

- `docker compose --profile dev up` starts the local development stack (gets the bind mount from the override)
- `docker compose --profile ci run --rm codecept` runs tests in CI (no override file, no bind mount, runs as `www-data`)
- `docker compose --profile tools run --rm quality` runs linting without starting the full stack

The `codecept` service sets `user: www-data` explicitly because in CI - without the bind mount - `/app` is owned by `www-data`, and the container must run as that user to write test output. Locally the bind mount replaces `/app` with host-owned files, so the `user: ${USERID}` from the base `php-fpm-test` service takes effect correctly.

---

## Exporting UID and GID from the Makefile

The `${USERID:-1000}` syntax in `compose.yaml` reads from the environment. The Makefile exports the values from the host at startup:

```makefile
export USERID  := $(shell id -u)
export GROUPID := $(shell id -g)
```

Any `make` target that invokes `docker compose` inherits these exports. The container runs as your actual host uid, so files created inside the container (migrations, generated code, test output) are owned by you on the host.

For running Compose commands directly (without `make`), export them in your shell first:

```bash
export USERID=$(id -u) GROUPID=$(id -g)
docker compose --profile dev up
```

---

## Mac vs Linux

On Linux, each container gets its own IP address. You can reach the web server directly at the container's IP - the port mapping in the override is optional.

On Mac, Docker Desktop runs containers inside a Linux VM. Container IPs are not reachable from the Mac host. The port mapping (`1080:80`) is required to access the dev server from a browser.

If your team is mixed, the `.dist` template includes the port mapping by default (it is harmless on Linux). Mac users may need additional overrides for volume performance - `:cached` or `:delegated` mount options, or switching to `bind` with `propagation: cached`.

---

## The Complete Setup

```
# commit these
compose.yaml
compose.override.dist.yaml

# gitignore this
compose.override.yaml
```

```makefile
# Makefile
export USERID  := $(shell id -u)
export GROUPID := $(shell id -g)

up:
    cp -n compose.override.dist.yaml compose.override.yaml  # idempotent first-run setup
    docker compose --profile dev up --detach
```

The `cp -n` flag (no-clobber) copies the template only if the override does not already exist. Running `make up` on a fresh clone sets everything up automatically.

---

### For AI agents

```
Docker Compose two-layer local dev pattern: commit compose.yaml (works in CI and prod) and compose.override.dist.yaml (template). Gitignore compose.override.yaml. Each developer runs: cp compose.override.dist.yaml compose.override.yaml. The override adds bind mount (./:/app replaces baked image code with live working copy) and port mappings. Compose auto-merges compose.override.yaml on top of compose.yaml when the file exists. CI never has the file - absence is the mechanism. The bind mount also fixes UID ownership: baked image files are owned by www-data (uid 33), but the container runs as the host user (uid 1000); the bind mount replaces /app with host-owned files. Export USERID/GROUPID from the Makefile via $(shell id -u) so compose.yaml can use user: ${USERID:-1000}. Compose profiles (dev/ci/prod/tools) select which services run; the override file controls how they are configured - they are independent mechanisms.
```

Reference: `https://michalsniezko.github.io/devops-infrastructure-cicd/compose-override-local-dev.html`
