# codebox

Run Claude Code, OpenCode, OpenAI Codex, or Polytoken in an isolated Docker container. Your project directory is mounted read-write, but the container has no access to the rest of your host filesystem.

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

`DIR` defaults to the current working directory. `ARGS` are passed through to `claude`, `opencode`, `codex`, or `polytoken`.

**Options:**

| Flag | Description |
|------|-------------|
| `-r`, `--rebuild` | Rebuild the Docker image(s), using the layer cache |
| `-R`, `--force-rebuild` | Rebuild the Docker image(s) without the layer cache |
| `--from IMAGE` | Build on `IMAGE` instead of `debian:trixie-slim`. The image needs `curl`, `bash` and `ca-certificates` |
| `--no-credentials` | Mount a separate sandbox config directory instead of your real one (see [Credentials](#credentials)) |
| `--mount HOST:CONTAINER` | Additional bind mount; repeatable. `:PATH` is shorthand for an identity mount (`:/data` = `/data:/data`), with options appended as `:/data:ro`. Also reads `CODEBOX_MOUNTS` from `.env.codebox` |
| `--no-gitconfig` | Do not mount your `~/.gitconfig` into the container |
| `--no-dockerfile` | Ignore the project's `Dockerfile.codebox` and run the base image instead |
| `--no-user-config` | Do not read the user-wide env file, `~/.config/codebox/env` |
| `--trust` | Use the project's `Dockerfile.codebox` and `.env.codebox` without the trust prompt (needed when there is no terminal, e.g. in scripts) |
| `--allow-home` | Skip the confirmation prompt when `DIR` or a `--mount` host path is your home directory or an ancestor of it |
| `--opencode` | Use [OpenCode 2](https://opencode.ai/v2/docs) instead of Claude Code |
| `--codex` | Use [OpenAI Codex](https://chatgpt.com/codex) instead of Claude Code |
| `--polytoken` | Use [Polytoken](https://polytoken.dev) instead of Claude Code |
| `--tailscale` | Put the container on your tailnet through a Tailscale sidecar container (see [Tailscale](#tailscale)) |
| `--serve PORT` | With `--tailscale`, make `PORT` in the container reachable from the tailnet over HTTP on port 80 |
| `--serve-https` | With `--serve`, use HTTPS on port 443 instead. The tailnet must have HTTPS certificates enabled |
| `--headless` | Run without a terminal, for scripts (see [Headless mode](#headless-mode)) |
| `--init` | Write a `Dockerfile.codebox` template into `DIR` |
| `-h`, `--help` | Show help |

`--opencode`, `--codex` and `--polytoken` are mutually exclusive.

**Examples:**

```sh
# Run Claude Code in the current directory
codebox

# Run OpenCode in the current directory
codebox --opencode

# Run OpenAI Codex in the current directory
codebox --codex

# Run Polytoken in the current directory
codebox --polytoken

# Run in a specific project
codebox ~/src/myproject

# Pass arguments to claude
codebox ~/src/myproject -- --model claude-opus-4-8

# Also mount a data directory read-only, at the same path
codebox --mount :/srv/data:ro ~/src/myproject

# Build on Ubuntu instead of Debian
codebox --from ubuntu:24.04 ~/src/myproject

# Maximum isolation: host login, CLAUDE.md and memory not exposed, auth via API key
ANTHROPIC_API_KEY=sk-... codebox --no-credentials ~/src/myproject

# The same isolation with a Claude subscription (token from `claude setup-token`)
CLAUDE_CODE_OAUTH_TOKEN=sk-ant-oat... codebox --no-credentials ~/src/myproject
```

## How it works

The first run builds a base image. Its Dockerfile is inside the `codebox` script, and it installs the chosen tool on top of `debian:trixie-slim` (or the `--from` image):

| Tool | Base image | With `--from IMAGE` |
|------|------------|---------------------|
| Claude Code (default) | `codebox` | `codebox--base-<hash of IMAGE>` |
| OpenCode (`--opencode`) | `codebox-opencode` | `codebox-opencode--base-<hash>` |
| Codex (`--codex`) | `codebox-codex` | `codebox-codex--base-<hash>` |
| Polytoken (`--polytoken`) | `codebox-polytoken` | `codebox-polytoken--base-<hash>` |

The container then runs with these settings:

- It runs as your host user (`uid:gid`), so files written inside are owned by you. `HOME` is `/home/sandbox`.
- Your project directory is mounted read-write at the same path as on the host, and is the working directory.
- All Linux capabilities are dropped (`--cap-drop ALL`), and processes cannot gain new privileges (`no-new-privileges`).
- A small init process runs as PID 1 (`--init`), with the agent under it. This is what lets the agent suspend itself (see [Suspending with Ctrl-Z](#suspending-with-ctrl-z)).
- Your `~/.gitconfig` is mounted read-only, so commits use your identity. Pass `--no-gitconfig` to skip it.
- These variables are forwarded from your environment when set: `ANTHROPIC_API_KEY`, `ANTHROPIC_BASE_URL`, `CLAUDE_CONFIG_DIR` and `CLAUDE_CODE_OAUTH_TOKEN` for Claude Code; `ANTHROPIC_API_KEY` and `ANTHROPIC_BASE_URL` for OpenCode; `OPENAI_API_KEY` and `OPENAI_BASE_URL` for Codex; `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `TERM` and `COLORTERM` for Polytoken (see [Polytoken](#polytoken)).
- Network access is not restricted. See [Security](#security).

The script exits with the agent's exit status.

If `DIR` is your home directory or contains it, codebox asks before mounting it, because the agent would get your SSH keys, cloud credentials and everything else under it. The same check applies to `--mount` and `CODEBOX_MOUNTS` host paths. Pass `--allow-home` to skip the prompt.

### Codex's own sandbox

Codex normally sandboxes the commands it runs with bubblewrap and Landlock. That needs an unprivileged user namespace, which codebox's container does not allow. The container is already the sandbox, so codebox runs Codex with `-s danger-full-access` to turn off the inner sandbox. Codex still asks for approval before it runs commands. Pass your own `-s`/`--sandbox`, `--full-auto` or `--dangerously-bypass-approvals-and-sandbox` to override this.

### Polytoken

Polytoken reads provider API keys from the variables that its provider config names, such as `${OPENAI_API_KEY}`. codebox forwards `ANTHROPIC_API_KEY` and `OPENAI_API_KEY` from your environment. Put any other key in the user-wide env file, `~/.config/codebox/env`. Logins made with `polytoken auth provider login` are kept in the mounted data directory, so they persist across runs.

Polytoken needs a terminal with at least 256 colours, and it checks `TERM` and `COLORTERM` to find out. `docker run -t` sets `TERM=xterm`, so codebox passes your `TERM` and `COLORTERM` through. If Polytoken still finds too few colours, set `POLYTOKEN_TUI_COLOR_MODE=256` or `truecolor` in the env file.

The Polytoken binary in the image belongs to root, so `polytoken update` cannot replace it. Run `codebox --polytoken -r` to install the newest version. The image sets `POLYTOKEN_SKIP_UPDATE_CHECK=1`, so Polytoken does not offer updates.

## Credentials

By default codebox mounts your real config directory read-write, so that login state and preferences persist across runs:

| Tool | Mounted | With `--no-credentials` |
|------|---------|-------------------------|
| Claude Code | `~/.claude` and `~/.claude.json` | `~/.claude-codebox` |
| OpenCode | `~/.config/opencode` and `~/.local/share/opencode` (provider logins, sessions and logs) | `~/.config/opencode-codebox` and `~/.local/share/opencode-codebox` |
| Codex | `~/.codex` (via `CODEX_HOME`, so it does not hide the Codex binary) | `~/.codex-codebox` |
| Polytoken | `~/.config/polytoken` and `~/.local/share/polytoken` (provider logins, sessions and logs) | `~/.config/polytoken-codebox` and `~/.local/share/polytoken-codebox` |

Mounting the real directory gives the agent your OAuth tokens. It also lets the agent change settings that your host installation reads, including Claude Code hooks, which run on the host. With `--no-credentials`, codebox mounts a separate directory instead. The sandbox keeps its own settings between runs, and your host login state is never exposed. Neither are your host `CLAUDE.md` and Claude Code memory, because both live in `~/.claude`. You can authenticate in three ways in that case:

- Log in inside the container (for Claude Code, `/login`). The login is saved in the sandbox directory, so it is kept between runs and is separate from your host login.
- Set `ANTHROPIC_API_KEY` (`OPENAI_API_KEY` for Codex).
- For a Claude subscription without logging in, run `claude setup-token` on the host and set the token it prints as `CLAUDE_CODE_OAUTH_TOKEN`. Put it in your environment or in the user-wide env file, `~/.config/codebox/env`. A project's `.env.codebox` may not set it.

## Suspending with Ctrl-Z

Ctrl-Z suspends the session and `fg` resumes it. The container keeps running while the session is suspended.

Claude Code handles Ctrl-Z itself. The first press shows a notice that Ctrl-Z now suspends and Ctrl+_ undoes input. Press Ctrl-Z again to suspend. Claude Code shows this notice every time, and its keybindings have no setting to turn it off.

When an agent does not handle Ctrl-Z itself, Docker's detach key catches it, and the first press suspends the session.

## Headless mode

`--headless` runs the container without a terminal, so you can use codebox from scripts, cron jobs and CI. codebox passes stdin to the agent, and the agent's output goes to stdout and stderr. codebox exits with the agent's exit status. Ctrl-Z job control is off in this mode.

codebox adds no flags to the agent's command, so give the agent's own non-interactive command:

```sh
# Claude Code
codebox --headless ~/src/myproject -- -p "Summarise the open TODOs"

# Codex
codebox --headless --codex ~/src/myproject -- exec "Summarise the open TODOs"

# OpenCode
codebox --headless --opencode ~/src/myproject -- run "Summarise the open TODOs"

# Input from a pipe
git diff | codebox --headless ~/src/myproject -- -p "Review this diff"
```

Nobody can answer permission prompts in headless mode. When a task needs to edit files or run commands, pass the agent's permission flags too, such as `--allowedTools` or `--dangerously-skip-permissions` for Claude Code.

codebox cannot ask for confirmation without a terminal. Pass `--trust` for a project whose `Dockerfile.codebox` or `.env.codebox` you have not trusted yet, and `--allow-home` when `DIR` is your home directory.

## Project-specific images

If a project directory contains a `Dockerfile.codebox`, codebox builds a project image on top of the base image. Use it to add project dependencies (compilers, runtimes, CLI tools) without changing the base image. `codebox --init` writes a commented template.

The FROM line is optional:

| `Dockerfile.codebox` starts with | Result |
|----------------------------------|--------|
| no FROM line | codebox adds `FROM` with the right base image |
| `FROM codebox`, `FROM codebox-opencode`, `FROM codebox-codex` or `FROM codebox-polytoken` | used as written |
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

These keys are refused, because they could redirect API traffic, along with the credentials it carries, or move the config directory: `ANTHROPIC_API_KEY`, `ANTHROPIC_BASE_URL`, `ANTHROPIC_AUTH_TOKEN`, `CLAUDE_CODE_OAUTH_TOKEN`, `CLAUDE_CONFIG_DIR`, `OPENAI_API_KEY`, `OPENAI_BASE_URL`, `CODEX_HOME`, `POLYTOKEN_CONFIG_PATH`, `POLYTOKEN_DATA_PATH`, `POLYTOKEN_CACHE_PATH`, `POLYTOKEN_DAEMON_CREDENTIAL_FILE`, `POLYTOKEN_OUTPOST_URL`, `POLYTOKEN_OUTPOST_TOKEN`, `XDG_CONFIG_HOME`, `XDG_DATA_HOME`, and `TS_EXTRA_ARGS` (see [Tailscale](#tailscale)). Polytoken (and OpenCode) read their config and data from under `XDG_CONFIG_HOME` and `XDG_DATA_HOME` when these are set.

`CODEBOX_MOUNTS` is a space-separated list of extra bind mounts in the same format as `--mount`:

```sh
CODEBOX_MOUNTS=":/srv/data:ro ../shared:/shared"
```

## User-wide environment: `~/.config/codebox/env`

`codebox` also reads `~/.config/codebox/env` (or `$XDG_CONFIG_HOME/codebox/env`) in the same format, before the project's `.env.codebox`. You own this file, so codebox uses it without a trust prompt, and it may set the keys that `.env.codebox` is refused. When both files set the same key, the project's value is used, unless the key is one of the refused keys. The mounts in `CODEBOX_MOUNTS` from both files are added together. This file is read by `codebox` only, not by `codebox-ts`, `codebox-tailclaude` or `codebox-web`. Pass `--no-user-config` to run without it.

## Duplicate mounts

Docker refuses to start a container when two mounts use the same container path, so `codebox` resolves duplicates before starting it:

- When two extra mounts (from the user env file, `.env.codebox` or `--mount`) use the same container path, the later one is used, in that order, so `--mount` wins. codebox prints a warning naming both mounts, unless the two are identical.
- An extra mount at a container path that codebox mounts itself (the project directory, the agent's config directory, `~/.gitconfig`) is ignored with a warning, because the agent needs those mounts.

## Trusting a project's files

`Dockerfile.codebox` and `.env.codebox` come from the project directory, which may be a repository you cloned. A `Dockerfile.codebox` runs arbitrary code at build time, and the image it builds can read the credentials mounted into the container. A `.env.codebox` can mount any host path into the container through `CODEBOX_MOUNTS`.

So before using either file for the first time, codebox shows a warning and asks whether to use it. It asks before building anything, including the base image named in a third-party FROM line. Your answer is remembered per directory and file content in `~/.config/codebox/trusted` (or `$XDG_CONFIG_HOME/codebox/trusted`). Editing the file makes codebox ask again.

Without a terminal, codebox refuses an untrusted file. Pass `--trust` to accept the project's files without asking.

## Tailscale

With `--tailscale`, codebox starts a `tailscale/tailscale` container named `codebox-tailscale-<hash>` next to the agent's container, and runs the agent's container in the sidecar's network namespace. The hash identifies the project directory. The tailnet node is named after the tool and the hash, for example `claude-1a2b3c4d`.

- Tailnet devices can reach ports that the agent's processes listen on. `--serve PORT` also publishes `PORT` through Tailscale Serve, over HTTP on port 80, or over HTTPS on port 443 with `--serve-https`. Each run clears the Serve settings of the previous run.
- The sidecar uses userspace networking, so it needs no tun device and no added capabilities, and tailnet addresses are not routed from inside the container. codebox adds the `tailscale` CLI to the agent's image (as an image named `<image>--tailscale`), so the agent can reach a tailnet device with `tailscale nc PEER PORT`.
- The node's state is kept in the docker volume `codebox-tailscale-state-<hash>`, so the node keeps its identity across runs. Remove the volume to remove the node's key from your machine.
- codebox removes the sidecar when the agent exits. Only one `--tailscale` session can run per project directory.
- If the sidecar cannot join the tailnet, for example because the auth key was rejected, codebox stops before starting the agent, prints the error from the sidecar's log, and saves the full log in `~/.local/state/codebox/tailscale-<hash>.log` (or under `$XDG_STATE_HOME`).

`TS_AUTH_KEY` (an [auth key](https://login.tailscale.com/admin/settings/keys)) and `TS_EXTRA_ARGS` (extra arguments for `tailscale up`) are read from your environment, then from the user-wide env file, then from `.env.codebox`, with later values overriding earlier ones. Both are passed to the sidecar only, never to the agent. `.env.codebox` may not set `TS_EXTRA_ARGS`, because a `--login-server` argument in it would send your auth key to another server. Without `TS_AUTH_KEY`, codebox prints a login URL and waits up to five minutes for you to log in.

```sh
TS_AUTH_KEY=tskey-auth-... codebox --tailscale --serve 3000 ~/src/myproject
```

## Security

Network access from the container is not restricted, so an agent can send your code or anything else it can read to any server. The tools are installed by piping their install scripts into a shell, so the image build trusts claude.ai, opencode.ai, chatgpt.com and get.polytoken.dev to serve the right scripts.

## Other scripts

This repository also contains `codebox-ts`, `codebox-tailclaude` and `codebox-web`, which add Tailscale access to the container. They are experimental, are not covered by this README, and lack some of the fixes in `codebox`. `codebox --tailscale` replaces `codebox-tailclaude`.
