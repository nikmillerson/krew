# Krew System Overview

## Purpose

Krew is a plugin manager for kubectl that enables users to discover, install, upgrade, and manage kubectl plugins in a consistent manner across different platforms (Linux, macOS, Windows).

## Core Architecture

### Directory Structure

Krew operates with a well-defined directory structure at `~/.krew/` (or `$KREW_ROOT` if overridden):

```
.krew/
├── bin/                    # Symlinks to plugin executables (must be in $PATH)
├── index/                  # Git repositories of plugin indexes
│   └── {index-name}/       # Each index is a separate git repo
│       └── plugins/        # Contains .yaml manifest files
├── store/                  # Installed plugin files
│   └── {plugin}/{version}/ # Versioned installations
└── receipts/               # Installation receipts (.yaml files)
    └── {plugin}.yaml       # Tracks installed version and source index
```

### Plugin Index System

- **Default Index**: `https://github.com/kubernetes-sigs/krew-index.git`
- **Custom Indexes**: Users can add additional indexes for private or custom plugins
- **Git-based**: Each index is a git repository, enabling:
  - Partial updates (git pull)
  - GPG verification
  - Rollback capability
  - Distributed operation

### Plugin Manifest Format

Each plugin has a YAML manifest file describing:
- **Metadata**: Name, version, description, homepage
- **Platform specifications**: OS/arch-specific binaries with:
  - Download URI and SHA256 checksum
  - File operations (how to extract and organize files)
  - Binary path (which file to symlink)
  - Platform selector (using Kubernetes label selectors)

### Installation Flow

1. **Index Update**: Git clone/pull index repositories
2. **Plugin Selection**: User specifies plugin(s) to install
3. **Platform Matching**: Find platform entry matching current OS/arch
4. **Download & Verify**: Fetch archive, verify SHA256
5. **Extract**: Unpack to temporary directory
6. **File Operations**: Execute manifest's file operations (move/copy)
7. **Symlink**: Create `kubectl-{plugin}` symlink in bin/
8. **Receipt**: Write installation receipt for tracking

### Plugin Naming

- **Simple name**: `plugin-name` (assumes default index)
- **Canonical name**: `index-name/plugin-name` (explicit index)
- **Display name**: Shows as `index-name/plugin-name` for non-default indexes

### Environment Variables

- **KREW_ROOT**: Override default ~/.krew location
- **KREW_NO_UPGRADE_CHECK**: Disable automatic upgrade notifications
- **KREW_BINARY**: Path to krew binary (used in tests)
- **KREW_OS/KREW_ARCH**: Override platform detection (testing only)

### User Invocation

Krew is invoked as a kubectl plugin:
```bash
kubectl krew [command] [args...]
```

All commands are implemented as subcommands under the root `krew` command using the Cobra framework.

### Bootstrap & Self-Installation

Krew itself is a krew plugin and can manage its own updates:
1. Initial krew binary is downloaded manually
2. Running `krew install krew` installs krew from the index
3. Subsequent updates use `krew upgrade krew`

### Pre-Run Checks

Before every command execution:
1. Check if `~/.krew/bin` is in $PATH (warn if not)
2. Ensure required directories exist
3. Run necessary migrations (receipts, index structure)
4. Occasionally check for krew updates (non-blocking, ~40% of runs)

### Security Model

- SHA256 verification for all downloads
- Plugin name validation (prevent path traversal)
- Security notices for default index plugins
- No automatic code execution during index update
- Custom indexes show security warnings

## System Constraints

1. **Single Binary**: Krew is distributed as a single binary
2. **No Runtime Dependencies**: Except git for index operations
3. **Cross-platform**: Consistent behavior on Linux, macOS, Windows
4. **Backward Compatibility**: Migrations handle breaking changes
5. **Idempotent Operations**: Re-running commands is safe

## Observable System States

1. **Uninitialized**: No indexes present
2. **Initialized**: At least one index cloned
3. **With Installations**: One or more plugins installed
4. **Up-to-date**: Index synchronized with remote
5. **Stale**: Index needs updating (visible in upgrade notifications)
