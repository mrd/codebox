# codebox

Run Claude Code, OpenCode, or OpenAI Codex in an isolated Docker container. Your project directory is mounted read-write, but the container has no access to the rest of your host filesystem.

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

`DIR` defaults to the current working directory. `ARGS` are passed through to `claude`, `opencode`, or `codex`.

**Options:**

| Flag | Description |
|------|-------------|
| `-r`, `--rebuild` | Force rebuild of the Docker image(s) |
| `--no-credentials` | Skip mounting credential/config directories; requires `ANTHROPIC_API_KEY` (or `OPENAI_API_KEY` for Codex) |
| `--no-dockerfile` | Ignore the project's `Dockerfile.codebox` and run the base image instead |
| `--mount HOST:CONTAINER` | Additional bind mount; repeatable. `:PATH` is shorthand for an identity mount (`:/data` = `/data:/data`), with options appended as `:/data:ro`. Also reads `CODEBOX_MOUNTS` from `.env.codebox` |
| `--allow-home` | Skip the confirmation prompt when the target directory is your home directory (or an ancestor of it) |
| `--opencode` | Use [OpenCode](https://opencode.ai) instead of Claude Code |
| `--codex` | Use [OpenAI Codex](https://chatgpt.com/codex) instead of Claude Code |
| `-h`, `--help` | Show help |

`--opencode` and `--codex` are mutually exclusive.

**Examples:**

```sh
# Run Claude Code in the current directory
codebox

# Run OpenCode in the current directory
codebox --opencode

# Run OpenAI Codex in the current directory
codebox --codex

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
- **OpenAI Codex** (`--codex`): base image `codebox-codex`, mounts your host `~/.codex` read-write (via `CODEX_HOME`) so login state (`auth.json`) and config persist across runs. (Codex installs its own binary under `~/.codex`, so codebox points `CODEX_HOME` at a separate mount to avoid shadowing it.) Codex normally sandboxes the commands it runs (bubblewrap/Landlock), which needs an unprivileged user namespace and can't work inside codebox's hardened container — so codebox runs Codex with `-s danger-full-access` to disable that redundant inner sandbox (the container is the sandbox; approval prompts stay on). Pass your own `-s`/`--sandbox`/`--full-auto`/`--dangerously-bypass-approvals-and-sandbox` to override.

In all cases the container runs as your host user (`uid:gid`), so files written inside are owned by you.

To avoid exposing credentials to the container, pass `--no-credentials` and authenticate via `ANTHROPIC_API_KEY` (or `OPENAI_API_KEY` for Codex) instead.

## Project-specific images

If a project directory contains a `Dockerfile.codebox`, codebox builds a project-specific image layered on top of the base image. Use this to add project dependencies (compilers, runtimes, CLI tools) without polluting the base image.

`Dockerfile.codebox` must start with the appropriate base image:

```dockerfile
FROM codebox          # for Claude Code (default)
FROM codebox-opencode # for --opencode
FROM codebox-codex    # for --codex
```

The project image is rebuilt automatically when the `Dockerfile.codebox` changes (detected by SHA-256). To force a rebuild, pass `-r`. To skip the project image entirely and run the base image, pass `--no-dockerfile`.

**Example** — adding Node.js to a project:

```dockerfile
FROM codebox

RUN apt-get update && apt-get install -y --no-install-recommends nodejs npm \
    && rm -rf /var/lib/apt/lists/*
```
