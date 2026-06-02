# codebox

Run Claude Code in an isolated Docker container. Your project directory is mounted read-write, but the container has no access to the rest of your host filesystem.

## Requirements

- Docker
- Bash

## Install

```sh
curl -fsSL https://raw.githubusercontent.com/mrd/codebox/main/codebox -o ~/.local/bin/codebox
chmod +x ~/.local/bin/codebox
```

Make sure `~/.local/bin` is in your `PATH`.

## Usage

```sh
codebox [OPTIONS] [DIR] [-- CLAUDE_ARGS...]
```

`DIR` defaults to the current working directory. `CLAUDE_ARGS` are passed through to `claude`.

**Options:**

| Flag | Description |
|------|-------------|
| `-r`, `--rebuild` | Force rebuild of the Docker image(s) |
| `-h`, `--help` | Show help |

**Examples:**

```sh
# Run Claude Code in the current directory
codebox

# Run in a specific project
codebox ~/src/myproject

# Pass arguments to claude
codebox ~/src/myproject -- --model claude-opus-4-8

```

## How it works

The first run builds a base Docker image (`codebox`) from the bundled `Dockerfile`. The image installs Claude Code on top of `debian:trixie-slim`.

Your `~/.claude` directory and `~/.claude.json` are mounted into the container so login state and preferences persist across runs. The container runs as your host user (`uid:gid`), so files written inside the container are owned by you.

## Project-specific images

If a project directory contains a `Dockerfile.codebox`, codebox builds a project-specific image layered on top of the base image. Use this to add project dependencies (compilers, runtimes, CLI tools) without polluting the base image.

`Dockerfile.codebox` must start with:

```dockerfile
FROM codebox
```

The project image is rebuilt automatically when the `Dockerfile.codebox` changes (detected by SHA-256). To force a rebuild, pass `-r`.

**Example** — adding Node.js to a project:

```dockerfile
FROM codebox

RUN apt-get update && apt-get install -y --no-install-recommends nodejs npm \
    && rm -rf /var/lib/apt/lists/*
```
