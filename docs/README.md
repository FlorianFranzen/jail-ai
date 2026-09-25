# jail-ai Documentation

## Guides

- **[jail-ai.1.md](jail-ai.1.md)** - the man page, in markdown (see below for the groff version)
- **[ADDING_AGENTS.md](ADDING_AGENTS.md)** - how to add support for a new AI agent
- **[cloud-layers.md](cloud-layers.md)** - AWS/GCP layers, pinned tool versions and how to update them
- **[cloud-layers-quick-ref.md](cloud-layers-quick-ref.md)** - condensed cheat sheet for the above
- **[EBPF_SETUP.md](EBPF_SETUP.md)** - installing the toolchain and helper for eBPF host blocking
- **[EBPF_SECURITY.md](EBPF_SECURITY.md)** - security architecture of the privileged helper
- **[EBPF_HELPER_MIGRATION.md](EBPF_HELPER_MIGRATION.md)** - migrating from in-process eBPF loading
- **[PERFORMANCE_OPTIMIZATIONS.md](PERFORMANCE_OPTIMIZATIONS.md)** - layer caching, parallel builds, prefetching
- **[TROUBLESHOOTING.md](TROUBLESHOOTING.md)** - common problems and their causes

## Design and implementation notes

`specs/` holds the design documents, one per feature:

- **[specs/IMAGE_TAGGING_STRATEGY.md](specs/IMAGE_TAGGING_STRATEGY.md)** - layer-based vs `--isolated` image tags
- **[specs/LAYERED_IMAGES_SUMMARY.md](specs/LAYERED_IMAGES_SUMMARY.md)** - the layered image system
- **[specs/UPGRADE_DETECTION_IMPLEMENTATION.md](specs/UPGRADE_DETECTION_IMPLEMENTATION.md)** - how outdated layers and image mismatches are detected
- **[specs/NIX_FLAKES_SUPPORT.md](specs/NIX_FLAKES_SUPPORT.md)** - Nix flakes, `--nix-store` modes, host-mode security
- **[specs/BLOCK_HOST_USAGE.md](specs/BLOCK_HOST_USAGE.md)** - eBPF host blocking, on by default
- **[specs/EBPF_IMPLEMENTATION.md](specs/EBPF_IMPLEMENTATION.md)** - eBPF program and loader internals
- **[specs/GIT_CONFIG_VERIFICATION_REPORT.md](specs/GIT_CONFIG_VERIFICATION_REPORT.md)** - git/GPG mapping verification
- **[specs/IMPLEMENTATION_SUMMARY.md](specs/IMPLEMENTATION_SUMMARY.md)** - overview of the implementation

## Man pages

Two files, which must be kept in sync:

- **jail-ai.1** - groff format, for the `man` command
- **jail-ai.1.md** - markdown, for web viewing

Preview without installing:

```bash
man ./docs/jail-ai.1
```

### Updating them

1. Edit **jail-ai.1** (groff)
2. Edit **jail-ai.1.md** (markdown)
3. Keep both in sync
4. Update the date in the `.TH` header (groff) and the footer (markdown)
5. Check it renders: `man ./docs/jail-ai.1`

### What they document

- All commands: `create`, `remove`, `status`, `save`, `agents`, `list`, `clean-all`, `upgrade`, `completions`
- Global options (`--verbose`, `--quiet`)
- Common options (`--backend`, `--image`, `--mount`, `--env`, `--memory`, `--cpu`, ...)
- Agent options (`--agent-configs`, `--git-gpg`, `--shell`, `--auth`, `--isolated`, `--nix-store`, ...)
- Examples, files and directories, environment variables
- Tools available in the default image
- Security considerations and best practices

Note that agent options belong **before** the agent name:
`jail-ai agents [OPTIONS] <AGENT> [-- ARGS...]`. Anything after the agent name is passed
through to the agent itself.

### Groff reference

- `.TH` - title header
- `.SH` - section header
- `.TP` - tagged paragraph (for options)
- `.B` - bold, `.I` - italic, `.BR` - bold/roman
- `.IP` - indented paragraph

## See Also

- [CLAUDE.md](../CLAUDE.md) - development guide and project overview
- [README.md](../README.md) - user-facing overview
- [Project Homepage](https://github.com/cyrinux/jail-ai)
