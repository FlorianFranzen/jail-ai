# Image Tagging Strategy

## Overview

jail-ai uses a **hybrid tagging strategy** that balances image sharing for efficiency with project isolation when needed.

## The Two Modes

### 1. Layer-Based Tagging (Default)

**Shared images** tagged by their layer composition for maximum reuse:

```
Project A (~/rust-project-a) → localhost/jail-ai-agent-claude:base-rust-nodejs
Project B (~/rust-project-b) → localhost/jail-ai-agent-claude:base-rust-nodejs
✅ Both projects share the same image (instant startup!)
```

### 2. Isolated Tagging (Opt-in with `--isolated`)

**Project-specific images** tagged with workspace hash for complete isolation:

```
Project A (~/rust-project-a) → localhost/jail-ai-agent-claude:abc12345
Project B (~/rust-project-b) → localhost/jail-ai-agent-claude:def67890
✅ Each project has its own unique image
```

## The Hybrid Architecture

All modes share the same foundation:
1. **Shared base/language layers**: Tagged with `:latest` (reused across all projects)
2. **Final image**: Tagged based on mode (layer-based by default, workspace hash with `--isolated`)

### Architecture

```
DEFAULT MODE (layer-based):
┌──────────────────────────────────────────────────────────┐
│  Shared: localhost/jail-ai-agent-claude:base-rust-nodejs│  ← All Rust+Claude projects
└──────────────────────────────────────────────────────────┘
                        ▲
                        │ builds from
                        │
┌──────────────────────────────────────────────────────────┐
│  Shared: localhost/jail-ai-nodejs:latest                 │  ← Shared language layers
│          localhost/jail-ai-rust:latest                   │
│          localhost/jail-ai-base:latest                   │
└──────────────────────────────────────────────────────────┘

ISOLATED MODE (--isolated):
┌──────────────────────────────────────────────────────┐
│  Project A: localhost/jail-ai-agent-claude:abc12345  │  ← Project-specific
│  Project B: localhost/jail-ai-agent-claude:def67890  │
└──────────────────────────────────────────────────────┘
                        ▲
                        │ builds from
                        │
┌──────────────────────────────────────────────────────┐
│  Shared: localhost/jail-ai-nodejs:latest             │  ← Shared language layers
│          localhost/jail-ai-rust:latest               │
│          localhost/jail-ai-base:latest               │
└──────────────────────────────────────────────────────┘
```

### Example: Two Rust Projects with Claude (Default Mode)

**Project A** (`~/rust-project-a`):
```bash
$ cd ~/rust-project-a
$ jail-ai agents claude

→ Detected: Rust
→ Building shared layers:
  ✓ localhost/jail-ai-base:latest (shared)
  ✓ localhost/jail-ai-rust:latest (shared)
  ✓ localhost/jail-ai-nodejs:latest (shared)
→ Using shared mode: layer-based image (base-rust-nodejs)
  ✓ localhost/jail-ai-agent-claude:base-rust-nodejs
→ Container uses: localhost/jail-ai-agent-claude:base-rust-nodejs
```

**Project B** (`~/rust-project-b`):
```bash
$ cd ~/rust-project-b
$ jail-ai agents claude

→ Detected: Rust
→ Reusing shared layers:
  ✓ localhost/jail-ai-base:latest (cached)
  ✓ localhost/jail-ai-rust:latest (cached)
  ✓ localhost/jail-ai-nodejs:latest (cached)
→ Using shared mode: layer-based image (base-rust-nodejs)
  ✓ localhost/jail-ai-agent-claude:base-rust-nodejs (cached, instant!)
→ Container uses: localhost/jail-ai-agent-claude:base-rust-nodejs
```

**Result**: Both projects share the same final image - instant startup for Project B!

### Example: Using Isolated Mode

When you need project-specific isolation:

```bash
$ cd ~/rust-project-a
$ jail-ai agents --isolated claude

→ Detected: Rust
→ Using isolated mode: workspace-specific image
→ Project hash (isolated mode): abc12345
→ Building shared layers:
  ✓ localhost/jail-ai-base:latest (shared)
  ✓ localhost/jail-ai-rust:latest (shared)
  ✓ localhost/jail-ai-nodejs:latest (shared)
→ Building agent image:
  ✓ localhost/jail-ai-agent-claude:abc12345
→ Container uses: localhost/jail-ai-agent-claude:abc12345
```

**When to use `--isolated`:**
- Testing different agent versions per project
- Project-specific customizations in final image
- Complete isolation from other projects
- Debugging image issues without affecting other projects

## Image Naming Convention

### Shared Layers (`:latest` tag)
These are **shared** across all projects for efficiency:

| Layer | Image Name | Purpose |
|-------|-----------|---------|
| Base | `localhost/jail-ai-base:latest` | Alpine + common tools |
| Language | `localhost/jail-ai-{rust,golang,python,nodejs,java}:latest` | Language toolchain |

### Project-Specific Images (`:project-hash` tag)
These are **unique** per project directory:

| Type | Image Name | Example |
|------|-----------|---------|
| Language only | `localhost/jail-ai-{lang}:{hash}` | `localhost/jail-ai-rust:abc12345` |
| With agent | `localhost/jail-ai-agent-{agent}:{hash}` | `localhost/jail-ai-agent-claude:abc12345` |

### Hash Generation

The project hash is derived from the **absolute path** of the workspace:

```rust
workspace: /home/user/my-rust-project
→ SHA256 hash of absolute path
→ Take first 8 characters: abc12345
→ Tag: localhost/jail-ai-agent-claude:abc12345
```

This ensures:
- ✅ **Reproducible**: Same directory = same hash
- ✅ **Unique**: Different directories = different hashes
- ✅ **Short**: 8 characters = readable + collision-resistant
- ✅ **Consistent**: Matches container naming (jail-myproject-abc12345-claude)

## Build Flow

### For Agent Commands (e.g., `jail-ai agents claude`)

```
1. Detect workspace → Generate hash (abc12345)
2. Detect project type → Rust
3. Build shared base:
   → localhost/jail-ai-base:latest
4. Build shared language:
   → localhost/jail-ai-rust:latest
5. Build shared nodejs (for agent):
   → localhost/jail-ai-nodejs:latest
6. Build project-specific agent:
   → localhost/jail-ai-agent-claude:abc12345
   → FROM localhost/jail-ai-nodejs:latest
   → RUN npm install -g @anthropic-ai/claude-code
7. Create container with final image
```

### For Language-Only Projects (e.g., `jail-ai create`)

```
1. Detect workspace → Generate hash (abc12345)
2. Detect project type → Rust
3. Build shared base:
   → localhost/jail-ai-base:latest
4. Build shared language:
   → localhost/jail-ai-rust:latest
5. Tag for project:
   → podman tag localhost/jail-ai-rust:latest \
                localhost/jail-ai-rust:abc12345
6. Create container with tagged image
```

## Benefits

### 1. **Efficient Storage**
Shared layers (base, language) are reused:
```
Project A: 
  - base:latest (200MB, shared)
  - rust:latest (300MB, shared)
  - agent-claude:abc12345 (50MB, unique)
  
Project B:
  - base:latest (reused, 0MB)
  - rust:latest (reused, 0MB)
  - agent-claude:def67890 (50MB, unique)

Total: 550MB (vs 600MB without sharing)
```

### 2. **Fast Rebuilds**
Only rebuild project-specific layer:
```bash
# Update agent for project A
$ cd ~/rust-project-a
$ jail-ai agents --upgrade claude

→ Reuses: base:latest, rust:latest, nodejs:latest
→ Rebuilds only: agent-claude:abc12345 (30 seconds)
```

### 3. **Perfect Isolation**
Each project gets its own final image:
```
Project A → agent-claude:abc12345
Project B → agent-claude:def67890
Project C → agent-copilot:78901234

All isolated, no conflicts! ✓
```

### 4. **Easy Cleanup**
Remove project-specific images without affecting others:
```bash
# Remove project A's image
$ podman rmi localhost/jail-ai-agent-claude:abc12345

# Projects B and C still work
# Shared layers (base, rust, nodejs) still cached
```

## Image Lifecycle

### Creation
```bash
$ jail-ai agents claude
→ Creates: jail-ai-agent-claude:abc12345
→ Creates: jail-myproject-abc12345-claude (container)
```

### Reuse
```bash
$ jail-ai agents claude  # Same project, same directory
→ Reuses: jail-ai-agent-claude:abc12345 (no rebuild)
→ Reuses: jail-myproject-abc12345-claude (if exists)
```

### Update
```bash
$ jail-ai agents --upgrade claude
→ Rebuilds: jail-ai-agent-claude:abc12345
→ Recreates: jail-myproject-abc12345-claude
```

### Cleanup
```bash
# Remove container and project-specific image
$ jail-ai remove
→ Removes: jail-myproject-abc12345-claude
→ Removes: jail-ai-agent-claude:abc12345

# Shared layers remain for other projects
```

## Example: Multi-Project Workflow

```bash
# Project 1: Rust + Claude
$ cd ~/rust-project-1
$ jail-ai agents claude
→ Builds: base, rust, nodejs
→ Creates: agent-claude:a1b2c3d4

# Project 2: Rust + Copilot (reuses base, rust, nodejs)
$ cd ~/rust-project-2
$ jail-ai agents copilot
→ Reuses: base, rust, nodejs
→ Creates: agent-copilot:e5f6g7h8

# Project 3: Python + Claude (reuses base, nodejs)
$ cd ~/python-project
$ jail-ai agents claude
→ Reuses: base, nodejs
→ Builds: python
→ Creates: agent-claude:i9j0k1l2

# List all images
$ podman images | grep jail-ai
jail-ai-base                latest    200MB
jail-ai-rust                latest    500MB
jail-ai-python              latest    280MB
jail-ai-nodejs              latest    250MB
jail-ai-agent-claude        a1b2c3d4  300MB  # Project 1
jail-ai-agent-copilot       e5f6g7h8  300MB  # Project 2
jail-ai-agent-claude        i9j0k1l2  330MB  # Project 3
```

## Matching Container Names

The image tags match the hash in container names:

```
Container: jail-myproject-abc12345-claude
Image:     localhost/jail-ai-agent-claude:abc12345
           └──────────────────┬──────────────────┘
                          Same hash!
```

This makes debugging easier:
```bash
$ podman ps
CONTAINER ID  IMAGE                                   NAME
...           jail-ai-agent-claude:abc12345           jail-myproject-abc12345-claude
```

## Summary

### Default Mode (Layer-Based Tagging)
✅ **Maximum reuse**: Projects with same layers share images instantly  
✅ **Fastest startup**: Zero build time for matching layer composition  
✅ **Storage efficient**: One image per unique layer stack  
✅ **Cross-project benefits**: Shared images improve all projects  
✅ **Simple management**: Fewer images to maintain  

### Isolated Mode (`--isolated` flag)
✅ **Complete isolation**: Each project has unique final image  
✅ **Project-specific**: Independent customization per workspace  
✅ **Safe testing**: Experiment without affecting other projects  
✅ **Consistent naming**: Image tag matches container hash  

### Both Modes Share
✅ **Efficient base layers**: `:latest` tag for language toolchains  
✅ **Fast layer builds**: Cached base/language layers  
✅ **Easy cleanup**: Remove final images safely  
✅ **Automatic detection**: Project type determines layer stack  

**Hybrid approach**: Default to sharing for speed, opt-in to isolation when needed! 🎯
