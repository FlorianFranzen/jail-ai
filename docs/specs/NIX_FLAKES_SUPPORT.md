# Nix Flakes Support for jail-ai

## Summary

Added automatic Nix flakes support to jail-ai. When a `flake.nix` file is detected in the workspace, jail-ai will automatically build and use a container image with Nix package manager and flakes enabled.

## Changes Made

### 1. Project Detection (`src/project_detection.rs`)
- Added `Nix` variant to `ProjectType` enum
- Added detection logic for `flake.nix` files
- Updated `language_layer()` to return `"nix"` for Nix projects
- Added test case for Nix project detection

### 2. Container Layer (`containerfiles/nix.Containerfile`)
The Nix development environment Containerfile:
- Builds on top of the base image
- Installs Nix in **single-user mode** (`--no-daemon`), owned by the `agent` user, with its
  profile in `/usr/local/nix-state` (outside the persistent `/home/agent` volume, so switching
  between nix and non-nix images never leaves stale Nix state in `$HOME`)
- Writes `/etc/nix/nix.conf`:
  ```
  experimental-features = nix-command flakes
  max-jobs = auto
  ```
  Nix defaults to `max-jobs = 1` and the single-user installer writes no `nix.conf`, so without
  this line derivations are built one at a time
- Snapshots the installed Nix closure into `/usr/local/nix-seed` (see *Nix Store* below)
- Adds `nix-wrapper` (enters `nix develop` when `/workspace/flake.nix` exists) and zsh/bash
  integration, all of which source `/usr/local/share/jail-ai/nix-init.sh`

### 3. Image Layers System (`src/image_layers.rs`)
- Added `NIX_IMAGE_NAME` constant: `localhost/jail-ai-nix:latest`
- Embedded `NIX_CONTAINERFILE` from the containerfiles directory
- Updated `get_language_image_name()` to handle Nix projects
- Updated `get_containerfile_content()` to include nix layer
- Updated `build_shared_layer()` to support nix image building
- Added test cases for Nix layer functionality

### 4. Documentation Updates
- Updated `CLAUDE.md` to mention Nix support in key features
- Updated `CLAUDE.md` to list Nix in custom image tools
- Updated `containerfiles/README.md` with Nix layer information
- Added Nix to the auto-detection table
- Added Nix image size estimate

## How It Works

### Automatic Detection
When you create a jail in a directory containing `flake.nix`:

```bash
cd /path/to/nix-project
jail-ai create my-nix-jail
```

jail-ai will:
1. Detect the `flake.nix` file
2. Build the base layer (if not cached)
3. Build the nix layer (if not cached): `localhost/jail-ai-nix:latest`
4. Tag it with project-specific hash
5. Start the jail with Nix available

### Using Nix in the Jail
Inside the jail, you can:

```bash
# Enter the jail
jail-ai join my-nix-jail

# Nix commands are available
nix --version

# Use flakes
nix flake show
nix develop     # Enter the flake's devShell
nix build       # Build the flake's default package
```

### With AI Agents
You can use Nix-enabled jails with AI agents:

```bash
# In a Nix project directory
jail-ai claude
```

This will create a jail with:
- Base layer
- Nix layer (for flake.nix support)
- Claude agent layer

## Nix Features

The Nix layer includes:
- **Nix Package Manager**: single-user install, no daemon inside the jail
- **Flakes Support**: enabled system-wide in `/etc/nix/nix.conf`
- **Parallel builds**: `max-jobs = auto`
- **Shell Integration**: automatic Nix environment setup in zsh and bash

## Nix Store

`/nix` is provided according to `--nix-store` (recorded in the `jail-ai.nix-store` label):

| Mode | `/nix` source | Notes |
|------|---------------|-------|
| `shared` (default) | global `jail-ai-nix` volume | all projects reuse downloaded and built paths |
| `project` | `<jail>__nix` volume per project | previous default; jails without the label are treated as `project` |
| `host` | host `/nix/store` (read-only) + host nix-daemon socket | `NIX_REMOTE=daemon`, the host nix client is put on `PATH`; builds run in the host daemon with your host user's trust level; Linux Nix hosts only |

Existing jails keep their mode across recreation (`--upgrade`) unless `--nix-store` is
given explicitly, which recreates the container when it differs.

Podman only populates a named volume from the image when the volume is empty, so a shared or
older volume may lack the Nix version of the current image. `nix-init.sh` checks this on
every shell and wrapper start (a single `stat`), and `jail-ai-nix-seed` then copies the missing
paths from `/usr/local/nix-seed`, registers them with `nix-store --load-db` and pins them with a
GC root in `/nix/var/nix/gcroots/jail-ai/`, so `nix-collect-garbage` in one jail never removes
another image's Nix. Removing a jail never removes the `jail-ai-nix` volume.

## Example Use Cases

### 1. Development with Nix Flakes
```bash
# Project with flake.nix
cd my-nix-project
jail-ai create dev-jail

# Inside jail
nix develop
# Your development environment is now loaded from the flake
```

### 2. Building Nix Projects
```bash
# Build a Nix flake project in isolation
jail-ai create build-jail
jail-ai exec build-jail -- nix build
```

### 3. AI Agent with Nix Project
```bash
# Let Claude help with your Nix project
cd nix-project
jail-ai claude "help me understand this flake.nix"
```

## Image Sizes

- **Base layer**: ~200MB (Alpine + common tools)
- **Nix layer**: ~350MB (base + Nix package manager)
- **Total**: ~350MB for Nix-enabled environment

Compare to installing Nix manually in every project: Nix layer is cached and reused!

## Testing

Added comprehensive tests:
- `test_detect_nix_project()` - Verifies flake.nix detection
- `test_get_language_image_name()` - Verifies correct image name for Nix
- `test_get_containerfile_content()` - Verifies nix Containerfile is embedded

Run tests with:
```bash
cargo test
```

## Future Enhancements

Potential improvements:
- Nix-specific resource limits
- Binary cache configuration
- Support for `shell.nix` (classic Nix shells)
- Nix channel management

## Notes

- Nix flakes are **experimental** but widely used in the Nix community
- There is no Nix daemon inside the jail; the `agent` user owns the store (or, with
  `--nix-store host`, the host daemon does)
- The Nix store persists in a volume across jail recreations (see *Nix Store*)

## Compatibility

- Works with all existing jail-ai features
- Compatible with resource limits, network isolation, etc.
- Can be combined with other project types (multi-language detection)
- Works with all AI agent integrations
