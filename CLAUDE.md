# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`jail-ai` is a Rust-based jail wrapper for sandboxing AI agents (Claude, Claude Code Router, Copilot, Cursor, Gemini, Codex, Jules) using podman. It provides isolation, resource limits, and workspace management for secure AI agent execution.

## Commands

### Build and Development

- **Build jail-ai**: `cargo build` or `make build`
- **Build release**: `cargo build --release`
- **Generate completions**: `make completions` (output: `dist/completions/`)
- **Build macOS bottle**: `make bottle-macos`
- **Run**: `cargo run -- <args>` or `make run ARGS="<args>"`
- **Run tests**: `cargo test` or `make test`
- **Run single test**: `cargo test <test_name>`
- **Lint**: `cargo clippy -- -D warnings` or `make clippy`
- **Format**: `cargo fmt` or `make fmt`
- **Update cloud versions**: `make update-cloud-versions` or `./scripts/update-cloud-versions.sh`
- **Rebuild cloud layers**: `make rebuild-cloud-layers`

### Container Image

- **Build custom AI agent image**: `make build-image`
  - Image includes: bash, ripgrep, cargo, go, node, python, and common dev tools
  - Default image name: `localhost/jail-ai-env:latest`
  - Customize with: `make build-image IMAGE_NAME=custom-name IMAGE_TAG=version`
- **Test image**: `make test-image`
- **Clean image**: `make clean`
- **Automatic building**: jail-ai will automatically build the layers it needs if they are not present
  - All Containerfiles are embedded in the binary (`include_str!` in `src/image_layers.rs`) and written
    to a temporary directory at build time
  - Layers are rebuilt automatically when their embedded definition changes, tracked via the
    `ai.jail.containerfile.hash` image label
  - To customize a single project, add a `jail-ai.Containerfile` to its root — see
    [Custom Project Layer](#custom-project-layer). There is no user-global Containerfile override
  - See `containerfiles/README.md` for what each layer contains

### Usage Examples

```bash
# The default image is automatically built if not present
# No need to run make build-image manually anymore!

# Create jail with auto-mounted workspace (uses default image, auto-builds if needed)
cargo run -- create my-agent

# Create jail with specific image
cargo run -- create my-agent --image alpine:latest

# Create jail without workspace mount
cargo run -- create my-agent --no-workspace

# Create jail with host networking (shares host's network namespace)
cargo run -- create my-agent --host-network

# Create jail with port mapping (e.g., PostgreSQL)
cargo run -- create my-agent -p 5432:5432

# Create jail with multiple port mappings
cargo run -- create my-agent -p 8080:80 -p 5432:5432

# Create jail with port mapping using UDP protocol
cargo run -- create my-agent -p 53:53/udp

# Create jail with every agent's config directory mounted
# (~/.claude, ~/.claude-code-router, ~/.config/.copilot, ~/.cursor, ~/.gemini,
#  ~/.coderabbit, ~/.codex, ~/.config/jules, ~/.config/opencode, ~/.pi)
cargo run -- create my-agent --agent-configs

# Create jail with Podman-in-Podman support (for running MCP agents)
cargo run -- create my-agent --podman

# Create jail with git and GPG configuration mapping
cargo run -- create my-agent --git-gpg

# Create jail with all agent configs and git/GPG support
cargo run -- create my-agent --agent-configs --git-gpg

# Create jail with custom workspace path
cargo run -- create my-agent --workspace-path /app

# Create jail skipping nix layer (when flake.nix is present, use other detected languages instead)
cargo run -- create my-agent --no-nix

# Nix store location (nix jails only): shared global volume (default), per-project volume, or host store via nix-daemon
cargo run -- create my-agent --nix-store shared
cargo run -- create my-agent --nix-store project
cargo run -- agents --nix-store host claude

# Create jail with cloud provider tools (AWS CLI + GCP gcloud)
cargo run -- agents --cloud claude

# Execute command in jail (non-interactive)
cargo run -- exec my-agent -- ls -la /workspace

# Execute command in jail (interactive shell)
cargo run -- exec my-agent --interactive -- bash

# Quick development jail
make dev-jail

# AI Agent commands with parameters (use -- to separate jail-ai params from agent params)
cargo run -- agents claude -- chat "help me debug this code"
cargo run -- agents claude -- --help
cargo run -- agents claude -- --version
# Claude Code Router - Automatically starts server with "ccr start" then runs "ccr code"
cargo run -- agents claude-code-router -- chat "help me debug this code"
cargo run -- agents copilot -- suggest "write tests"
cargo run -- agents gemini -- --model gemini-pro "explain this"
# CodeRabbit CLI - AI-powered code review assistant
cargo run -- agents coderabbit -- review
cargo run -- agents coderabbit -- --help
# Codex CLI - Open interactive shell for OAuth authentication
cargo run -- agents --auth codex

# Codex CLI - Run agent after authentication is complete
cargo run -- agents codex -- generate "create a REST API"
cargo run -- agents jules -- chat "help me debug this code"
cargo run -- agents jules -- --help

# AI Agent with host networking (full access to host network)
cargo run -- agents --host-network claude -- chat "help me with this service"

# AI Agent with port mapping (e.g., for connecting to host PostgreSQL)
cargo run -- agents -p 5432:5432 claude -- chat "help me with database queries"
cargo run -- agents -p 8080:80 -p 5432:5432 claude -- chat "debug my web app and database"

# AI Agent with Podman-in-Podman support (for running MCP agents inside the jail)
cargo run -- agents --podman claude -- chat "help me run containers"

# AI Agent commands skipping nix layer (use other detected languages instead)
cargo run -- agents --no-nix claude -- chat "help me debug this code"
cargo run -- agents --no-nix copilot -- suggest "write tests"

# Codex CLI with manual authentication (interactive shell)
cargo run -- agents --shell codex

# Start interactive shell in agent jail (without running the agent)
cargo run -- agents --shell claude
cargo run -- agents --shell claude-code-router
cargo run -- agents --shell copilot
cargo run -- agents --shell coderabbit
cargo run -- agents --shell jules

# Layer-based (shared) vs Isolated images
# By default, jail-ai uses layer-based tagging for image sharing across projects
cargo run -- agents claude  # Uses shared image: localhost/jail-ai-agent-claude:base-rust-nodejs

# Use --isolated flag for project-specific images (workspace hash)
cargo run -- agents --isolated claude  # Uses isolated image: localhost/jail-ai-agent-claude:abc12345
```

### Container Upgrade Detection

When you re-enter an existing container, jail-ai automatically checks for updates in two areas:

1. **Outdated Layers** - Detects if layer images need rebuilding (e.g., after upgrading jail-ai binary)
2. **Container Image Mismatch** - Detects if the container's image differs from what should be used

This ensures a smooth experience after upgrading your jail-ai binary or when Containerfiles are updated.

**Example prompt when updates are detected:**

```
🔄 Update available for your jail environment!

📦 Outdated layers detected:
  • base
  • rust
  • agent-claude

This typically happens after upgrading the jail-ai binary.
Layers contain updated tools, dependencies, or security patches.

🐳 Container image mismatch:
  Current:  localhost/jail-ai-agent-claude:base-rust-nodejs-abc123
  Expected: localhost/jail-ai-agent-claude:base-rust-nodejs-def456

💡 Recommendation: Use --upgrade to:
  • Rebuild outdated layers with latest definitions
  • Recreate container with the correct image
  • Ensure you have the latest tools and security patches

Your data in /home/agent will be preserved during the rebuild.

Would you like to rebuild now? (y/N):
```

**How it works:**

- The check is automatic when entering an existing container (no `--upgrade` needed)
- Compares embedded Containerfile hashes to detect outdated layers
- Compares the container's current image with what should be used
- If you choose to rebuild (type `y`), it performs a full `--upgrade` automatically
- If you decline (type `N` or just press Enter), the existing container continues to run
- Your data in `/home/agent` is preserved via persistent volumes during rebuilds

**To force a rebuild without prompting:**

```bash
cargo run -- agents --upgrade claude
```

**Common scenarios:**

- **After upgrading jail-ai binary**: Embedded Containerfiles change, so layers are detected as outdated
- **After `git pull` with Containerfile changes**: Layers with modified definitions are detected
- **After rebuilding specific layers with `--upgrade`**: Container image tag changes, prompting recreation

### Version Management

- Version is managed in `Cargo.toml` and should follow semantic versioning
- Auto-bump version when making changes according to semver rules

## Code Style

- Prefer functional programming patterns in Rust
- Add debug logging where appropriate
- Ensure clippy passes without errors
- Add and update tests as you progress through changes

## Architecture

- **backend/**: Trait-based abstraction with podman implementation
- **cli.rs**: CLI interface using clap
- **config.rs**: Jail configuration with serialization support
- **jail.rs**: High-level jail manager with builder pattern
- **error.rs**: Error types and Result alias

## Key Features

- **Custom Development Image**: Pre-built container with bash, ripgrep, cargo, go, node, python, nix, and essential dev tools
- **AI Agent Integration**: Claude Code, Claude Code Router, GitHub Copilot CLI, Cursor Agent, Gemini CLI, Codex CLI, and Jules CLI pre-installed
- **Nix Flakes Support**: When `flake.nix` is detected, Nix takes precedence and only base + nix + agent layers are used (excluding rust/node/etc). Use `--no-nix` to skip nix and activate other language layers instead
- **Nix Parallel Builds**: `/etc/nix/nix.conf` sets `max-jobs = auto` (Nix itself defaults to 1 and the single-user installer writes no config)
- **Nix Store Sharing** (`--nix-store`): `shared` (default) mounts the global `jail-ai-nix` volume in every nix jail; `project` keeps one `<jail>__nix` volume per project (legacy behaviour, assumed for jails without the `jail-ai.nix-store` label); `host` mounts the host `/nix/store` read-only plus the nix-daemon socket (`NIX_REMOTE=daemon`). Existing jails keep their mode unless the flag is passed. `jail-ai-nix-seed` restores the image's Nix into volumes that lack it and pins it with a GC root. The shared volume is never removed with a jail
- **Automatic Upgrade Detection**: When re-entering an existing container, jail-ai automatically checks for outdated layers and container image mismatches, prompting you to rebuild. This ensures a smooth experience after upgrading the jail-ai binary.
- **Workspace Auto-mounting**: Current working directory is automatically mounted to `/workspace` in the jail (configurable)
- **Environment Inheritance**: Automatically inherits `TERM` and timezone (`TZ`) from host environment, sets `EDITOR=vim`, and configures `SSH_AUTH_SOCK` when GPG SSH agent socket is available
- **Agent Config Mounting**: running an agent mounts that agent's own config directory (e.g. `~/.claude` → `/home/agent/.claude`) — the whole directory, not just credentials. Use `--agent-configs` to mount *every* agent's directory instead of only the one being run
- **Opt-in Git/GPG Mapping**: Use `--git-gpg` to enable git configuration (name, email, signing key) and GPG config (`~/.gnupg`) mounting
- **Podman Backend**: Uses podman for OCI container management
- **Resource Limits**: Memory and CPU quota restrictions
- **Network Isolation**: Configurable network access (disabled, private, or shared)
- **Bind Mounts**: Support for read-only and read-write mounts

## Custom Project Layer

jail-ai supports project-specific customization through a `jail-ai.Containerfile` in your project root. When this file is present, it will be automatically detected and built as a custom layer in the image stack:

**Build Order**: base → language layers (rust, nodejs, etc.) → **custom** → agent layers (claude, copilot, etc.)

### Creating a Custom Layer

Create a file named `jail-ai.Containerfile` in your project root:

```dockerfile
# jail-ai.Containerfile - Custom layer for this project
ARG BASE_IMAGE
FROM ${BASE_IMAGE}

USER root

# Install project-specific tools
RUN apt-get update && apt-get install -y --no-install-recommends \
    vim-nox \
    && rm -rf /var/lib/apt/lists/*

# Install project-specific npm packages
RUN npm install -g your-package

USER agent
WORKDIR /workspace
```

### Features

- **Automatic Detection**: jail-ai automatically detects `jail-ai.Containerfile` in the project root
- **Layer Caching**: The custom layer is cached and only rebuilt when the Containerfile changes or with `--upgrade`
- **Shared by Default**: In shared mode, projects with the same language stack + custom layer share the image
- **Isolated Mode**: Use `--isolated` flag for project-specific images (uses workspace hash in tag)
- **Force Rebuild**: Use `--upgrade --layers custom` to force rebuild of just the custom layer

### Use Cases

- **Project-specific tools**: Install tools that are only needed for this project
- **Custom configurations**: Set up project-specific environment variables or configs
- **Version pinning**: Install specific versions of tools that differ from the base layers
- **Development dependencies**: Add debugging tools or profilers for this project
- **CI/CD alignment**: Match the exact environment used in your CI/CD pipeline

### Image Tag Examples

Without custom layer: `localhost/jail-ai-agent-claude:base-rust-nodejs`  
With custom layer: `localhost/jail-ai-agent-claude:base-rust-nodejs-custom`

See `examples/jail-ai.Containerfile` for a complete template.

## Custom Image Tools

The layered image system automatically detects your project type and builds appropriate images with the following tools:

- **Shell**: zsh (default with Powerlevel10k theme), bash
- **Shell Enhancements**:
  - **fzf** - Fuzzy finder for command history and file search
  - **Powerlevel10k** - Beautiful and fast zsh theme with git integration
- **Search**: ripgrep, fd-find
- **Languages**: 
  - Rust (cargo, clippy, rustfmt)
  - Go (go toolchain)
  - Node.js (npm, yarn, pnpm)
  - Python 3 (pip, black, pylint, mypy, pytest)
  - Java (OpenJDK, Maven, Gradle)
  - Nix (with flakes support)
  - PHP (8.2, Composer, PHPUnit, PHPStan, PHP-CS-Fixer)
  - C/C++ (GCC, Clang, CMake, vcpkg, GDB, Valgrind)
  - C# (.NET SDK 8.0, dotnet-format, EF Core tools)
  - AWS (AWS CLI v2, eksctl, SAM CLI, CDK, Session Manager, cfn-lint, rain, Copilot CLI, Steampipe) - **all versions pinned** for efficient caching
  - GCP (gcloud CLI, Cloud SQL Proxy, Skaffold, kubectl, Helm, kpt, emulators) - **all versions pinned** for efficient caching

> **📌 Cloud Layers Optimization**: AWS and GCP layers use pinned package versions to avoid unnecessary rebuilds. Layers only rebuild when you update versions in the Containerfiles. See [docs/cloud-layers.md](docs/cloud-layers.md) for details on updating tools.
- **Build tools**: gcc, make, cmake, pkg-config
- **Utilities**: git, vim, nano, helix, jq, tree, tmux, htop, gh (GitHub CLI)
- **Python tools**: black, pylint, mypy, pytest, uv/uvx (for running MCP servers)
- **Rust tools**: clippy, rustfmt
- **Nix tools**: Nix package manager with flakes enabled
- **AI Coding Agents**:
  - **Claude Code** (`claude`) - Anthropic's CLI coding assistant
  - **Claude Code Router** (`ccr`) - Routes Claude Code requests to different models (OpenRouter, DeepSeek, Ollama, Gemini, etc.)
    - **Note**: Claude Code Router automatically starts a server with `ccr start` in the background before executing `ccr code` with your command. This is handled transparently by jail-ai.
  - **GitHub Copilot CLI** (`copilot`) - GitHub's AI pair programmer
  - **Cursor Agent** (`cursor-agent`) - Cursor's terminal AI agent
  - **Gemini CLI** (`gemini`) - Google's AI terminal assistant
  - **CodeRabbit CLI** (`coderabbit`) - AI-powered code review assistant
  - **Codex CLI** (`codex`) - OpenAI's Codex CLI for code generation
  - **Jules CLI** (`jules`) - Google's AI coding assistant CLI

### AI Agent Authentication

The AI coding agents require authentication.

**Default behavior:** running an agent mounts that agent's own config directory into the jail. This
happens automatically; there is no per-agent flag for it.

| Agent | Mounted from the host |
|-------|------------------------|
| `claude` | `~/.claude` → `/home/agent/.claude` |
| `claude-code-router` | `~/.claude` and `~/.claude-code-router` |
| `coderabbit` | `~/.coderabbit` |
| `codex` | `~/.codex` |
| `copilot` | `~/.config/.copilot` |
| `cursor` | `~/.cursor` and `~/.config/cursor` |
| `gemini` | `~/.gemini` |
| `jules` | `~/.config/jules` |
| `opencode` | `~/.config/opencode` |
| `pi` | `~/.pi` |

> **Note on scope:** this mounts the *whole* directory, not just credentials. For Claude Code that
> includes `settings.json`, conversation history and MCP server config, all readable and writable by
> the agent. `jail-ai create` on its own (no agent) mounts none of this.

**`--agent-configs`** mounts *every* agent's directory from the table above, rather than only the one
you are running — useful when a jail needs more than one agent's credentials.

**OAuth agents.** `coderabbit`, `codex` and `jules` authenticate through a browser flow rather than a
credentials file, so they need a shell in the container:

```bash
jail-ai agents --auth coderabbit   # then run: coderabbit auth
jail-ai agents --auth codex        # then run: codex auth login
jail-ai agents --auth jules        # then run: jules auth
```

`--auth` joins the container if it is running, or starts it if stopped. It also switches the jail to
host networking so the OAuth redirect can reach your browser, so **restart without `--auth` when you
are done** to restore network isolation.

jail-ai detects a missing or empty credential file for these three on first run and enables auth mode
by itself, so you usually do not need to pass `--auth` explicitly.

Example aliases:

```bash
# Each agent's own config directory is mounted automatically
alias jail-claude='jail-ai agents claude'
alias jail-ccr='jail-ai agents claude-code-router'
alias jail-copilot='jail-ai agents copilot'
alias jail-cursor='jail-ai agents cursor'
alias jail-gemini='jail-ai agents gemini'
alias jail-coderabbit='jail-ai agents coderabbit'
alias jail-codex='jail-ai agents codex'
alias jail-jules='jail-ai agents jules'

# Add git/GPG signing, or every agent's config rather than just Claude's
alias jail-claude-git='jail-ai agents --git-gpg claude'
alias jail-claude-all='jail-ai agents --agent-configs --git-gpg claude'
```

### Git and GPG Configuration Mapping

When `--git-gpg` flag is used, jail-ai will:

**Git Configuration:**

1. **Local Git Config**: If a `.git/config` file exists in the current directory, it will be mounted to `/home/agent/.gitconfig` in the jail
2. **Project Git Config Fallback**: If no local git config file exists, it will read your project's git configuration (or global as fallback) and create a `.gitconfig` file in the container with the following values:
   - `user.name`, `user.email`, `user.signingkey`
   - `commit.gpgsign`, `tag.gpgsign` - Enables automatic GPG signing for commits and tags
   - `gpg.format`, `gpg.program`, `gpg.ssh.allowedsignersfile` - GPG configuration
   - `core.editor`, `init.defaultbranch`, `pull.rebase`, `push.autosetupremote` - Git behavior settings

**GPG Configuration:**

- Mount your `~/.gnupg` directory to `/home/agent/.gnupg` in the jail
- This allows GPG signing to work inside the jail using your host's GPG keys
- **GPG Agent Sockets**: Automatically mounts all GPG agent sockets from `/run/user/<UID>/gnupg/` including:
  - `S.gpg-agent` - Main GPG agent socket
  - `S.gpg-agent.ssh` - SSH authentication socket (sets `SSH_AUTH_SOCK` environment variable)
  - `S.gpg-agent.extra` - Extra GPG agent socket
  - `S.gpg-agent.browser` - Browser GPG agent socket
- **SSH-based GPG Signing**: If `gpg.format=ssh` is configured, automatically mounts your SSH allowed signers file (`gpg.ssh.allowedsignersfile`) to `/home/agent/.ssh/allowed_signers` in the jail
  - If the SSH allowed signers file doesn't exist, a warning is logged but the jail creation continues
  - SSH GPG signing may not work properly without the allowed signers file
  - Supports both quoted and unquoted git config values (e.g., `"ssh"` or `ssh`)

This ensures that git commits and tags made inside the jail will use your configured identity and signing key.

**Note**: Git and GPG configuration mapping are **opt-in** (disabled by default). Use `--git-gpg` flag to enable them.

## Podman-in-Podman Support

jail-ai supports running containers inside the jail using Podman-in-Podman. This is useful for MCP (Model Context Protocol) agents that need to spawn containers.

### Usage

Use the `--podman` flag to enable Podman-in-Podman support:

```bash
# Create a jail with Podman-in-Podman support
cargo run -- create my-agent --podman

# Run Claude with Podman-in-Podman support
cargo run -- agents --podman claude

# Combine with other options
cargo run -- agents --podman --git-gpg claude
```

### How it Works

When `--podman` is enabled, jail-ai:

1. **Mounts the Podman socket**: The host's Podman socket (`/run/user/<UID>/podman/podman.sock`) is mounted to `/run/podman/podman.sock` inside the container
2. **Sets CONTAINER_HOST**: The `CONTAINER_HOST` environment variable is set to `unix:///run/podman/podman.sock` for seamless Podman remote operation

### Security Considerations

- The container can manage containers on the host through the socket
- This is more secure than using `--privileged` as it doesn't grant full root access
- Containers started from within the jail run on the host, not nested
- Use with caution in multi-tenant environments

### Use Cases

- **MCP Agents**: Run Model Context Protocol agents that need to spawn containers (e.g., for isolated code execution)
- **CI/CD Tasks**: Run build tasks that require container operations
- **Development Workflows**: Test containerized applications from within the jail

### Example with MCP Agent

```bash
# Start Claude with Podman support for MCP agents
jail-ai agents --podman claude

# Inside the jail, you can now use podman commands
podman run --rm alpine echo "Hello from nested container"
```

## Shell Features

The container uses **zsh** as the default shell with:

- **Powerlevel10k (p10k)** - Fast, minimal theme with git integration
- **fzf** integration for command history search (Ctrl+R) and fuzzy completion
- **Smart history** - 10000 entries with deduplication and sharing
- **Useful aliases** - `ll` for detailed listing, colored ripgrep

FZF keybindings:

- `Ctrl+R` - Search command history
- `Ctrl+T` - Search files in current directory
- `Alt+C` - Change to subdirectory

## Git Workflow

- Use conventional commits with emoji to distinguish commit types
- Use `git add -p` for selective staging when appropriate
- Auto-commit when it makes sense (completed features, fixed bugs, etc.)
- you have to build before commit
- Never modify my git config.
