# codebox

Run Claude Code or OpenCode in an isolated Docker container. Your project directory is mounted read-write, but the container has no access to the rest of your host filesystem.

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
codebox [OPTIONS] [DIR] [-- ARGS...]
```

`DIR` defaults to the current working directory. `ARGS` are passed through to `claude` or `opencode`.

**Options:**

| Flag | Description |
|------|-------------|
| `-r`, `--rebuild` | Force rebuild of the Docker image(s) |
| `--no-credentials` | Skip mounting credential/config directories; requires `ANTHROPIC_API_KEY` |
| `--no-dockerfile` | Ignore the project's `Dockerfile.codebox` and run the base image instead |
| `--opencode` | Use [OpenCode](https://opencode.ai) instead of Claude Code |
| `-h`, `--help` | Show help |

**Examples:**

```sh
# Run Claude Code in the current directory
codebox

# Run OpenCode in the current directory
codebox --opencode

# Run in a specific project
codebox ~/src/myproject

# Pass arguments to claude
codebox ~/src/myproject -- --model claude-opus-4-8

# Maximum isolation: no OAuth token exposed, auth via API key
ANTHROPIC_API_KEY=sk-... codebox --no-credentials ~/src/myproject

# OpenCode with API key only (no config dir mounted)
ANTHROPIC_API_KEY=sk-... codebox --opencode --no-credentials ~/src/myproject
```

## How it works

The first run builds a base Docker image from the bundled `Dockerfile`. The image installs the chosen tool on top of `debian:trixie-slim`.

- **Claude Code** (default): base image `codebox`, mounts `~/.claude` and `~/.claude.json` read-write so login state and preferences persist across runs.
- **OpenCode** (`--opencode`): base image `codebox-opencode`, mounts `~/.config/opencode` read-write so configuration persists across runs.

In both cases the container runs as your host user (`uid:gid`), so files written inside are owned by you.

To avoid exposing credentials to the container, pass `--no-credentials` and authenticate via `ANTHROPIC_API_KEY` instead.

## Project-specific images

If a project directory contains a `Dockerfile.codebox`, codebox builds a project-specific image layered on top of the base image. Use this to add project dependencies (compilers, runtimes, CLI tools) without polluting the base image.

`Dockerfile.codebox` must start with the appropriate base image:

```dockerfile
FROM codebox          # for Claude Code (default)
FROM codebox-opencode # for --opencode
```

The project image is rebuilt automatically when the `Dockerfile.codebox` changes (detected by SHA-256). To force a rebuild, pass `-r`. To skip the project image entirely and run the base image, pass `--no-dockerfile`.

**Example** — adding Node.js to a project:

```dockerfile
FROM codebox

RUN apt-get update && apt-get install -y --no-install-recommends nodejs npm \
    && rm -rf /var/lib/apt/lists/*
```
