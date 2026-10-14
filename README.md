# codebox

Run Claude Code, OpenCode, or OpenAI Codex in an isolated Docker container. Your project directory is mounted read-write, but the container has no access to the rest of your host filesystem.

## Requirements

- Docker
- Bash
- GNU coreutils (`realpath`, `sha256sum`, `mktemp --suffix`), so in practice Linux

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
| `-r`, `--rebuild` | Rebuild the Docker image(s), using the layer cache |
| `-R`, `--force-rebuild` | Rebuild the Docker image(s) without the layer cache |
| `--from IMAGE` | Build on `IMAGE` instead of `debian:trixie-slim`. The image needs `curl`, `bash` and `ca-certificates` |
| `--no-credentials` | Mount a separate sandbox config directory instead of your real one (see [Credentials](#credentials)); authenticate with `ANTHROPIC_API_KEY` (or `OPENAI_API_KEY` for Codex) |
| `--mount HOST:CONTAINER` | Additional bind mount; repeatable. `:PATH` is shorthand for an identity mount (`:/data` = `/data:/data`), with options appended as `:/data:ro`. Also reads `CODEBOX_MOUNTS` from `.env.codebox` |
| `--no-gitconfig` | Do not mount your `~/.gitconfig` into the container |
| `--no-dockerfile` | Ignore the project's `Dockerfile.codebox` and run the base image instead |
| `--trust` | Use the project's `Dockerfile.codebox` and `.env.codebox` without the trust prompt (needed when there is no terminal, e.g. in scripts) |
| `--allow-home` | Skip the confirmation prompt when `DIR` or a `--mount` host path is your home directory or an ancestor of it |
| `--opencode` | Use [OpenCode](https://opencode.ai) instead of Claude Code |
| `--codex` | Use [OpenAI Codex](https://chatgpt.com/codex) instead of Claude Code |
| `--init` | Write a `Dockerfile.codebox` template into `DIR` |
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

# Also mount a data directory read-only, at the same path
codebox --mount :/srv/data:ro ~/src/myproject

# Build on Ubuntu instead of Debian
codebox --from ubuntu:24.04 ~/src/myproject

# Maximum isolation: no OAuth token exposed, auth via API key
ANTHROPIC_API_KEY=sk-... codebox --no-credentials ~/src/myproject
```

## How it works

The first run builds a base image. Its Dockerfile is inside the `codebox` script, and it installs the chosen tool on top of `debian:trixie-slim` (or the `--from` image):

| Tool | Base image | With `--from IMAGE` |
|------|------------|---------------------|
| Claude Code (default) | `codebox` | `codebox--base-<hash of IMAGE>` |
| OpenCode (`--opencode`) | `codebox-opencode` | `codebox-opencode--base-<hash>` |
| Codex (`--codex`) | `codebox-codex` | `codebox-codex--base-<hash>` |

The container then runs with these settings:

- It runs as your host user (`uid:gid`), so files written inside are owned by you. `HOME` is `/home/sandbox`.
- Your project directory is mounted read-write at the same path as on the host, and is the working directory.
- All Linux capabilities are dropped (`--cap-drop ALL`), and processes cannot gain new privileges (`no-new-privileges`).
- A small init process runs as PID 1 (`--init`), with the agent under it. This is what lets the agent suspend itself (see [Suspending with Ctrl-Z](#suspending-with-ctrl-z)).
- Your `~/.gitconfig` is mounted read-only, so commits use your identity. Pass `--no-gitconfig` to skip it.
- These variables are forwarded from your environment when set: `ANTHROPIC_API_KEY`, `ANTHROPIC_BASE_URL` and `CLAUDE_CONFIG_DIR` for Claude Code; `ANTHROPIC_API_KEY` and `ANTHROPIC_BASE_URL` for OpenCode; `OPENAI_API_KEY` and `OPENAI_BASE_URL` for Codex.
- Network access is not restricted. See [Security](#security).

The script exits with the agent's exit status.

If `DIR` is your home directory or contains it, codebox asks before mounting it, because the agent would get your SSH keys, cloud credentials and everything else under it. The same check applies to `--mount` and `CODEBOX_MOUNTS` host paths. Pass `--allow-home` to skip the prompt.

### Codex's own sandbox

Codex normally sandboxes the commands it runs with bubblewrap and Landlock. That needs an unprivileged user namespace, which codebox's container does not allow. The container is already the sandbox, so codebox runs Codex with `-s danger-full-access` to turn off the inner sandbox. Codex still asks for approval before it runs commands. Pass your own `-s`/`--sandbox`, `--full-auto` or `--dangerously-bypass-approvals-and-sandbox` to override this.

## Credentials

By default codebox mounts your real config directory read-write, so that login state and preferences persist across runs:

| Tool | Mounted | With `--no-credentials` |
|------|---------|-------------------------|
| Claude Code | `~/.claude` and `~/.claude.json` | `~/.claude-codebox` |
| OpenCode | `~/.config/opencode` | `~/.config/opencode-codebox` |
| Codex | `~/.codex` (via `CODEX_HOME`, so it does not hide the Codex binary) | `~/.codex-codebox` |

Mounting the real directory gives the agent your OAuth tokens. It also lets the agent change settings that your host installation reads, including Claude Code hooks, which run on the host. With `--no-credentials`, codebox mounts a separate directory instead. The sandbox keeps its own settings between runs, and your host login state is never exposed. Authenticate with `ANTHROPIC_API_KEY` (or `OPENAI_API_KEY` for Codex) in that case.

## Suspending with Ctrl-Z

Ctrl-Z suspends the session and `fg` resumes it. The container keeps running while the session is suspended.

Claude Code handles Ctrl-Z itself. The first press shows a notice that Ctrl-Z now suspends and Ctrl+_ undoes input. Press Ctrl-Z again to suspend. Claude Code shows this notice every time, and its keybindings have no setting to turn it off.

When an agent does not handle Ctrl-Z itself, Docker's detach key catches it, and the first press suspends the session.

## Project-specific images

If a project directory contains a `Dockerfile.codebox`, codebox builds a project image on top of the base image. Use it to add project dependencies (compilers, runtimes, CLI tools) without changing the base image. `codebox --init` writes a commented template.

The FROM line is optional:

| `Dockerfile.codebox` starts with | Result |
|----------------------------------|--------|
| no FROM line | codebox adds `FROM` with the right base image |
| `FROM codebox`, `FROM codebox-opencode` or `FROM codebox-codex` | used as written |
| `FROM` any other image, e.g. `FROM ubuntu:24.04` | same as `--from ubuntu:24.04`: codebox builds a base image on top of it, and points the FROM line at that base image |

`--from` takes priority over the FROM line. Flags such as `--platform` and an `AS stage` suffix on the FROM line are kept.

If the file declares `ARG USER_ID` or `ARG GROUP_ID`, codebox passes your uid and gid as build arguments. Use them to give files installed at build time to the user the container runs as:

```dockerfile
ARG USER_ID
ARG GROUP_ID
RUN chown -R $USER_ID:$GROUP_ID /opt/android-sdk
```

The project image is rebuilt when `Dockerfile.codebox` changes (detected by SHA-256), and when the base image has been rebuilt since the project image was built. Pass `-r` to force a rebuild, or `--no-dockerfile` to skip the project image and run the base image.

**Example:** adding Node.js to a project:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends nodejs npm \
    && rm -rf /var/lib/apt/lists/*
```

## Project environment: `.env.codebox`

If the project directory contains a `.env.codebox`, each `KEY=VALUE` line is passed to the container as an environment variable. Lines starting with `#` are ignored. Values are used as written, so quotes are kept as part of the value (`CODEBOX_MOUNTS` is the exception). codebox prints the name of each variable it passes on.

These keys are refused, because they could redirect API traffic, along with the credentials it carries, or move the config directory: `ANTHROPIC_API_KEY`, `ANTHROPIC_BASE_URL`, `ANTHROPIC_AUTH_TOKEN`, `CLAUDE_CONFIG_DIR`, `OPENAI_API_KEY`, `OPENAI_BASE_URL`, `CODEX_HOME`.

`CODEBOX_MOUNTS` is a space-separated list of extra bind mounts in the same format as `--mount`:

```sh
CODEBOX_MOUNTS=":/srv/data:ro ../shared:/shared"
```

## Trusting a project's files

`Dockerfile.codebox` and `.env.codebox` come from the project directory, which may be a repository you cloned. A `Dockerfile.codebox` runs arbitrary code at build time, and the image it builds can read the credentials mounted into the container. A `.env.codebox` can mount any host path into the container through `CODEBOX_MOUNTS`.

So before using either file for the first time, codebox shows a warning and asks whether to use it. It asks before building anything, including the base image named in a third-party FROM line. Your answer is remembered per directory and file content in `~/.config/codebox/trusted` (or `$XDG_CONFIG_HOME/codebox/trusted`). Editing the file makes codebox ask again.

Without a terminal, codebox refuses an untrusted file. Pass `--trust` to accept the project's files without asking.

## Security

Network access from the container is not restricted, so an agent can send your code or anything else it can read to any server. The tools are installed by piping their install scripts into a shell, so the image build trusts claude.ai, opencode.ai and chatgpt.com to serve the right scripts.

## Other scripts

This repository also contains `codebox-ts`, `codebox-tailclaude` and `codebox-web`, which add Tailscale access to the container. They are experimental, are not covered by this README, and lack some of the fixes in `codebox`.
