# JAIL-AI(1) User Commands

## NAME

jail-ai - AI Agent Jail Manager for sandboxing AI agents using podman

## SYNOPSIS

```
jail-ai [-v|--verbose] [-q|--quiet] COMMAND [OPTIONS]
jail-ai create [NAME] [OPTIONS]
jail-ai remove [NAME] [-f|--force] [-v|--volume]
jail-ai status [NAME]
jail-ai save [NAME] -o|--output FILE
jail-ai agents [OPTIONS] AGENT [-- ARGS...]
jail-ai list [-c|--current]
jail-ai clean-all [-f|--force] [-v|--volume]
jail-ai upgrade [NAME] [--all] [OPTIONS]
```

## DESCRIPTION

**jail-ai** is a Rust-based jail wrapper for sandboxing AI agents (Claude, Copilot, Cursor, Gemini) using podman. It provides isolation, resource limits, and workspace management for secure AI agent execution.

The tool automatically builds and manages custom container images with development tools, handles workspace mounting, manages authentication credentials, and provides granular control over resource limits and network access.

Use AI agent commands (e.g., `jail-ai agents claude`, `jail-ai agents copilot`) to quickly start agents in jails, or use `jail-ai create` to set up custom jails.

## COMMANDS

### create [NAME] [OPTIONS]

Create a new jail with the specified name. If no name is provided, generates a unique name based on the current directory path. The jail will have the current working directory auto-mounted to /workspace unless `--no-workspace` is specified.

### remove [NAME] [OPTIONS]

Remove (stop and delete) a jail. If no name is provided, operates on the jail associated with the current directory. Use `-f` to skip confirmation prompts and `-v` to also remove persistent volumes.

### status [NAME]

Show the status of a jail including its running state, resource usage, and configuration. If no name is provided, shows status for the current directory's jail.

### save [NAME] -o FILE

Save the jail configuration to a JSON file. This can be used to recreate the jail later with identical settings.

### agents [OPTIONS] AGENT [-- ARGS...]

Run an AI coding agent in a jail for the current directory. **AGENT** is one of `claude`,
`claude-code-router`, `coderabbit`, `codex`, `copilot`, `cursor`, `gemini`, `jules`, `opencode` or
`pi`.

The agent's own config directory is mounted automatically (`~/.claude` for `claude`,
`~/.config/.copilot` for `copilot`, and so on) — the whole directory, not just the credentials. Use
`--agent-configs` to mount every agent's directory instead of only the one being run.

For agents that authenticate through a browser (`coderabbit`, `codex`, `jules`), use `--auth` to open
an interactive shell in the container; it joins a running container or starts a stopped one, and
switches the jail to host networking so the OAuth redirect can reach your browser. Restart without
`--auth` afterwards to restore network isolation.

All options must appear **before** AGENT. Anything after `--` is passed through to the agent command
unchanged.

### list [-c|--current]

List all jails managed by jail-ai. Use `-c` to show only jails associated with the current directory.

### clean-all [-f|--force] [-v|--volume]

Stop and remove all jail-ai containers. Use `-f` to skip confirmation and `-v` to also remove persistent volumes.

### upgrade [NAME] [OPTIONS]

Upgrade a jail by recreating it with the latest image version. Use `--all` to upgrade all jails. Use `-i` to specify a different image. The jail's configuration and mounts are preserved.

## GLOBAL OPTIONS

| Option | Description |
|--------|-------------|
| `-v`, `--verbose` | Enable verbose logging with DEBUG level output. Shows detailed information about operations and backend commands. |
| `-q`, `--quiet` | Quiet mode - suppress INFO logs, only show warnings and errors. Conflicts with `--verbose`. |

## COMMON OPTIONS

The following options are available for the **create** and **agents** commands:

| Option | Description |
|--------|-------------|
| `-b`, `--backend BACKEND` | Backend type (only 'podman' is supported, kept for compatibility). Default: podman |
| `-i`, `--image IMAGE` | Base container image to use. Default: `localhost/jail-ai-env:latest`. The default image is automatically built if not present. |
| `-m`, `--mount SOURCE:TARGET[:ro]` | Add a bind mount. Can be specified multiple times. Append ':ro' for read-only mounts. Example: `-m /host/data:/data:ro` |
| `-e`, `--env KEY=VALUE` | Set environment variable in the jail. Can be specified multiple times. Example: `-e DEBUG=1` |
| `--no-network` | Disable network access for the jail. The container will not have any network connectivity. |
| `--memory MB` | Memory limit in megabytes. Example: `--memory 2048` (for 2GB limit) |
| `--cpu PERCENT` | CPU quota percentage (0-100). Example: `--cpu 50` (for 50% of one CPU core) |
| `--no-workspace` | Skip auto-mounting the current working directory to /workspace in the jail. |
| `--workspace-path PATH` | Custom workspace path inside jail. Default: `/workspace`. Example: `--workspace-path /app` |
| `--agent-configs` | Mount all agent config directories. Mounts every agent's config directory rather than only the one being run. Useful when working with multiple AI agents in the same jail. |
| `--git-gpg` | Enable git and GPG configuration mapping. Mounts `~/.gnupg` directory, all GPG agent sockets (`/run/user/<UID>/gnupg/*`), and creates or mounts git configuration with user identity and signing settings. If `gpg.format=ssh` is configured, also mounts the SSH allowed signers file. This is opt-in (disabled by default) for security. |
| `--layers LAYER[,LAYER...]` | Force specific image layers (comma-separated). Available layers: base, rust, python, nodejs, golang, java, php, cpp, csharp, nix, kubernetes, terraform, and agent-specific layers (agent-claude, agent-copilot, agent-cursor, agent-gemini, agent-codex). Example: `--layers base,rust,python` |
| `--shell` | Start an interactive shell instead of running the agent command. This allows you to use the jail environment without executing the AI agent. Example: `jail-ai agents --shell claude` |
| `--no-nix` | Ignore flake.nix file and skip nix layer if present. By default, jail-ai automatically detects and builds nix layer when flake.nix is found in the workspace. Use this flag to disable nix detection and layer building. |
| `--nix-store MODE` | Where `/nix` lives for Nix jails. `shared` (default for new jails): one global `jail-ai-nix` volume reused by all projects. `project`: one `<jail>__nix` volume per project (the previous default). `host`: bind-mount the host's `/nix/store` read-only and build through the host nix-daemon (requires a Nix host). **Security:** the jail inherits your host user's nix-daemon trust level, so if that user is in the daemon's `trusted-users` the jail can set substituters and build hooks and import paths into the host store. See `docs/specs/NIX_FLAKES_SUPPORT.md`. Existing jails keep their mode unless this flag is given. |

## EXAMPLES

### Basic Usage

Create a jail with auto-mounted workspace (uses default image, auto-builds if needed):
```bash
jail-ai create my-agent
```

Create a jail with specific image:
```bash
jail-ai create my-agent --image alpine:latest
```

Create a jail without workspace mount:
```bash
jail-ai create my-agent --no-workspace
```

Create a jail ignoring flake.nix file (skip nix layer):
```bash
jail-ai create my-agent --no-nix
```

Execute command in jail (non-interactive):
```bash
jail-ai exec my-agent -- ls -la /workspace
```

### AI Agent Usage

Quick start Claude Code (minimal auth - only API keys):
```bash
jail-ai agents claude
```

Start Claude with full config directory and git/GPG support:
```bash
jail-ai agents --git-gpg claude
```

Start GitHub Copilot with authentication:
```bash
jail-ai agents copilot
```

Start Cursor Agent with authentication:
```bash
jail-ai agents cursor
```

Start Gemini CLI with authentication:
```bash
jail-ai agents gemini
```

Start Codex CLI with API key authentication:
```bash
jail-ai agents codex
```

Pass arguments to the AI agent (including flags with hyphens):
```bash
jail-ai agents claude -- chat "help me debug this code"
jail-ai agents claude -- --help
jail-ai agents claude -- --version
jail-ai agents copilot -- suggest "write tests"
jail-ai agents gemini -- --model gemini-pro "explain this code"
```

Start an interactive shell in an agent jail (without running the agent):
```bash
jail-ai agents --shell claude
jail-ai agents --shell copilot
```

AI agent commands ignoring flake.nix file (skip nix layer):
```bash
jail-ai agents --no-nix claude -- chat "help me debug this code"
jail-ai agents --no-nix copilot -- suggest "write tests"
```

### Configuration Mounting

Start jail with all agent configs and git/GPG support:
```bash
jail-ai create my-agent --agent-configs --git-gpg
```

Start Claude with custom workspace path:
```bash
jail-ai agents --workspace-path /app claude
```

### Resource Limits

Create jail with memory and CPU limits:
```bash
jail-ai create my-agent --memory 2048 --cpu 50
```

Create jail without network access:
```bash
jail-ai create my-agent --no-network
```

### Custom Mounts and Environment

Create jail with custom bind mounts:
```bash
jail-ai create my-agent \
  --mount /host/data:/data:ro \
  --mount /host/config:/config
```

Create jail with custom environment variables:
```bash
jail-ai create my-agent \
  --env DEBUG=1 \
  --env API_KEY=secret
```

### Image Management

Rebuild outdated layers and recreate the container:
```bash
jail-ai create my-agent --upgrade
```

Create jail with specific language layers:
```bash
jail-ai create my-agent --layers base,rust,python,nodejs
```

Upgrade all jails to latest image:
```bash
jail-ai upgrade --all
```

Upgrade specific jail to new image:
```bash
jail-ai upgrade my-agent --image localhost/jail-ai-env:v2
```

### Management Commands

List all jails:
```bash
jail-ai list
```

List jails for current directory:
```bash
jail-ai list --current
```

Check jail status:
```bash
jail-ai status my-agent
```

Save jail configuration to file:
```bash
jail-ai save my-agent --output config.json
```

Remove a jail:
```bash
jail-ai remove my-agent
```

Remove jail with volumes (force):
```bash
jail-ai remove my-agent --force --volume
```

Clean up all jails:
```bash
jail-ai clean-all --force
```

## FILES

| File/Directory | Description |
|----------------|-------------|
| `~/.claude/` | Claude Code configuration directory (settings, commands, history). Mounted for `agents claude`. |
| `~/.config/.copilot/` | GitHub Copilot CLI configuration directory. Mounted for `agents copilot`. |
| `~/.cursor/` | Cursor Agent data directory. Mounted for `agents cursor`. |
| `~/.config/cursor/` | Cursor Agent configuration directory. Mounted for `agents cursor`. |
| `~/.gemini/` | Gemini CLI configuration directory. Mounted for `agents gemini`. |
| `~/.codex/` | Codex CLI configuration directory. Mounted for `agents codex`. |
| `~/.gnupg/` | GPG configuration directory. Mounted when `--git-gpg` is specified, enabling GPG signing inside the jail. |
| `/run/user/<UID>/gnupg/` | GPG agent socket directory. All sockets (S.gpg-agent, S.gpg-agent.ssh, S.gpg-agent.extra, S.gpg-agent.browser) are mounted when `--git-gpg` is specified. |
| `.git/config` | Local git configuration. If present, mounted to /home/agent/.gitconfig when `--git-gpg` is specified. Otherwise, git configuration is extracted from the project or global config. |
| `~/.ssh/allowed_signers` | SSH allowed signers file for GPG SSH signing. Mounted when `--git-gpg` is specified and gpg.format=ssh is configured. |

## ENVIRONMENT

jail-ai automatically configures the following environment variables in the jail:

| Variable | Description |
|----------|-------------|
| `TERM` | Inherited from the host environment for proper terminal emulation. |
| `TZ` | Timezone inherited from the host environment. |
| `EDITOR` | Set to 'vim' by default. |
| `SSH_AUTH_SOCK` | Configured when the GPG SSH agent socket is available and `--git-gpg` is specified. Points to `/run/user/<UID>/gnupg/S.gpg-agent.ssh` for SSH authentication via GPG. |

## IMAGE TOOLS

The default jail-ai-env image includes the following tools and languages:

### Shell and Shell Enhancements
- **zsh** (default shell with Powerlevel10k theme)
- **bash**
- **fzf** - Fuzzy finder for command history (Ctrl+R), file search (Ctrl+T), and directory change (Alt+C)
- **Powerlevel10k** - Beautiful and fast zsh theme with git integration

### Search and Navigation Tools
- **ripgrep** (rg) - Fast text search
- **fd-find** - Fast file search

### Programming Languages
- **Rust** (cargo, clippy, rustfmt)
- **Go** (go toolchain)
- **Node.js** (npm, yarn, pnpm)
- **Python 3** (pip, black, pylint, mypy, pytest)
- **Java** (OpenJDK, Maven, Gradle)
- **Nix** (with flakes support, automatic detection)
- **PHP** (8.2, Composer, PHPUnit, PHPStan, PHP-CS-Fixer)
- **C/C++** (GCC, Clang, CMake, vcpkg, GDB, Valgrind)
- **C#** (.NET SDK 8.0, dotnet-format, EF Core tools)

### Build Tools
- gcc, make, cmake, pkg-config

### Utilities
- git, vim, nano, helix
- jq, tree, tmux, htop
- gh (GitHub CLI)

### AI Coding Agents
- **Claude Code** (claude) - Anthropic's CLI coding assistant
- **GitHub Copilot CLI** (copilot) - GitHub's AI pair programmer
- **Cursor Agent** (cursor-agent) - Cursor's terminal AI agent
- **Gemini CLI** (gemini) - Google's AI terminal assistant
- **Codex CLI** (codex) - OpenAI's Codex CLI for code generation

## NOTES

### Backend Support
Currently, only podman is supported as the backend. The `--backend` option is kept for compatibility but has no effect.

### Automatic Image Building
Images are built automatically if not present when creating a jail or running an agent. All Containerfiles are embedded in the binary and written to a temporary directory at build time; to customize a single project, add a `jail-ai.Containerfile` to its root.

### Jail Naming
Jail names are automatically generated from the current directory path using a hash for uniqueness. Names are sanitized to match podman requirements (`[a-zA-Z0-9][a-zA-Z0-9_.-]*`).

### Security Considerations
- **Authentication mounting** is minimal by default: Claude only auto-mounts API credentials (`~/.claude/.credentials.json`), other agents require explicit flags.
- **Git and GPG configuration mounting** is opt-in (use `--git-gpg`) for security.
- Use `--no-network` for maximum isolation when network access is not needed.
- **Resource limits** (`--memory`, `--cpu`) help prevent runaway processes.
- **Read-only mounts** (`-m source:target:ro`) prevent accidental modifications.

### Nix Flakes Support
When a `flake.nix` file is detected in the workspace, jail-ai automatically loads the Nix development environment using 'nix develop' when entering the jail.

Nix is configured with `max-jobs = auto`, so independent derivations are built in parallel (Nix itself defaults to `max-jobs = 1`).

By default all Nix jails share the `jail-ai-nix` store volume, so paths downloaded or built in one project are reused in the others. Removing a jail never removes this volume; use `nix-collect-garbage` inside a jail or `podman volume rm jail-ai-nix` to reclaim space. Use `--nix-store host` on a Nix host to reuse the host's store directly; note that this grants the jail your host user's nix-daemon trust level, and that `max-jobs = auto` no longer applies because the host daemon schedules the builds.

## AUTHORS

Cyril Levis <git@levis.name>

## COPYRIGHT

Copyright © 2025 Cyril Levis

License: MIT OR Apache-2.0

## SEE ALSO

**podman**(1), **podman-run**(1), **podman-exec**(1), **systemd-nspawn**(1)

**Project homepage:** https://github.com/cyrinux/jail-ai

**Documentation:** https://docs.rs/jail-ai

---

*jail-ai 0.31.0 - 2025-10-09*
