# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Krew is the package manager for kubectl plugins. It helps users discover, install, and manage kubectl plugins in a consistent way across platforms (Linux, macOS, Windows). Krew itself is distributed as a kubectl plugin and uses a Git-based index system to track available plugins.

## Build and Test Commands

### Building
```bash
# Build krew binary for current platform (output: out/bin/krew-{os}_{arch})
hack/make-binary.sh

# Build for all supported platforms
hack/make-binaries.sh
```

### Testing
```bash
# Run all unit tests and code quality checks (boilerplate, tests, linter, patterns)
hack/run-tests.sh

# Run unit tests only (with race detection)
go test -short -race sigs.k8s.io/krew/...

# Run a specific test
go test -v -run TestName sigs.k8s.io/krew/pkg/...

# Run integration tests (requires building binary first)
hack/make-binary.sh
hack/run-integration-tests.sh

# Run specific integration test
hack/make-binary.sh
hack/run-integration-tests.sh -test.v -test.run TestFoo
```

### Code Quality
```bash
# Run linter (golangci-lint)
hack/run-lint.sh

# Format Go code with goimports (required before commits)
goimports -local sigs.k8s.io/krew -w cmd pkg integration_test

# Verify boilerplate license headers
hack/verify-boilerplate.sh

# Verify code patterns
hack/verify-code-patterns.sh
```

### Testing in Sandbox
```bash
# Test krew without affecting your actual krew installation
mkdir playground
KREW_ROOT="$PWD/playground" ./out/bin/krew-{os}_{arch} update

# Or use Docker sandbox
hack/run-in-docker.sh
```

## Architecture Overview

### Core Directory Structure

- **`cmd/krew/`**: Main krew CLI application entry point and command implementations
  - `cmd/krew/main.go`: Entry point that defers to cobra command tree
  - `cmd/krew/cmd/`: Cobra command implementations (install, upgrade, search, etc.)
  - `cmd/krew/cmd/internal/`: Internal helpers for commands (warnings, setup checks, tag fetching)

- **`internal/`**: Private application code
  - `environment/`: Path management for krew's directory structure (`~/.krew/`)
  - `installation/`: Core plugin installation, upgrade, and platform detection logic
  - `download/`: HTTP download, verification (SHA256), and extraction (tar.gz, zip)
  - `index/`: Plugin index operations (scanning, validation) and custom index support
  - `gitutil/`: Git operations for index repository management
  - `indexmigration/` & `receiptsmigration/`: Migration logic for version upgrades

- **`pkg/`**: Public API packages
  - `constants/`: API version, index URI, manifest extension
  - `index/`: Plugin manifest types (Plugin, Platform, FileOperation, Receipt)

- **`integration_test/`**: High-level integration tests that test full krew workflows

### Key Architectural Concepts

#### 1. Krew Directory Structure (`~/.krew/` or `$KREW_ROOT`)
```
.krew/
├── bin/                    # Symlinks to plugin executables (add to $PATH)
├── index/                  # Git repositories of plugin indexes
│   └── default/            # Default krew-index repo
│       └── plugins/        # .yaml manifest files
├── store/                  # Installed plugin files
│   └── {plugin}/{version}/ # Versioned plugin installations
└── receipts/               # Installation receipts (.yaml files)
    └── {plugin}.yaml       # Tracks installed version and source index
```

The `Paths` struct in `internal/environment/environment.go` centralizes all path logic.

#### 2. Plugin Manifest Format
Plugin manifests (`pkg/index/types.go`) describe how to install plugins:
- **Platform-specific**: Each `Platform` entry has URI, SHA256, file operations, binary path, and OS/arch selector
- **File operations**: Specify how to move files from archive to installation directory (mimics `mv` command)
- **Versioning**: Supports semver for upgrade checks

#### 3. Installation Flow
1. **Index Update** (`cmd/krew/cmd/update.go`): Git clone/pull index repositories
2. **Plugin Installation** (`internal/installation/install.go`):
   - Select platform matching current OS/arch using label selectors
   - Download archive from URI and verify SHA256
   - Extract to temp directory
   - Execute file operations (move files per manifest)
   - Create symlink in `bin/` directory
   - Write receipt to track installation
3. **Upgrade** (`internal/installation/upgrade.go`): Compare installed version with index version using semver

#### 4. Self-Installation & Bootstrap
Krew is itself a krew plugin. It bootstraps by:
1. Downloading krew binary to temp location
2. Running `krew update` (clone index)
3. Running `krew install krew` (install itself from index)
4. Cleaning up temp binary

This is reflected in the architecture where krew dogfoods its own installation mechanism.

#### 5. Index System
- **Default index**: `https://github.com/kubernetes-sigs/krew-index.git` (cloned to `~/.krew/index/default/`)
- **Custom indexes**: Users can add additional indexes (stored in `~/.krew/index/{name}/`)
- **Git-based**: Enables partial updates, GPG verification, and rollbacks
- **Manifest files**: Each plugin has a `.yaml` file (e.g., `foo.yaml`) in `plugins/` directory

#### 6. Cobra Command Structure
Commands are organized in `cmd/krew/cmd/`:
- `root.go`: Handles PATH checks, directory setup, migrations, upgrade notifications
- Individual command files: `install.go`, `upgrade.go`, `search.go`, `list.go`, `uninstall.go`, etc.
- Persistent pre-run hook checks for migrations and ensures krew's bin directory is in PATH

## Go Module Structure

- Module path: `sigs.k8s.io/krew`
- Go version: 1.25
- Key dependencies:
  - `github.com/spf13/cobra` - CLI framework
  - `k8s.io/apimachinery` - Kubernetes label selectors for platform matching
  - `k8s.io/client-go` - Kubernetes client utilities
  - `sigs.k8s.io/yaml` - YAML parsing for manifests

## Code Style Requirements

- **Go formatting**: Use `goimports` with `-local sigs.k8s.io/krew` to group imports correctly
- **Shell formatting**: Scripts are formatted with `shfmt -w -i=2`
- **Boilerplate**: All source files must have Apache 2.0 license header
- **Testing**: All new code should be covered by tests
- **Import order**: Standard library, third-party, then local `sigs.k8s.io/krew` imports

## Platform Considerations

### macOS Development
The tools in `hack/` expect GNU binaries. Install via Homebrew:
```bash
brew install coreutils grep gnu-sed
export PATH=$(brew --prefix coreutils)/libexec/gnubin:$PATH
export PATH="$(brew --prefix grep)/libexec/gnubin:$PATH"
export PATH="$(brew --prefix gnu-sed)/libexec/gnubin:$PATH"
```

### Windows
- Krew includes special cleanup logic for stale installations on Windows
- Installation symlinks are handled differently (see `internal/installation/`)

## Important Environment Variables

- **`KREW_ROOT`**: Override default `~/.krew` location (useful for testing)
- **`KREW_NO_UPGRADE_CHECK`**: Disable automatic upgrade checks
- **`KREW_BINARY`**: Specify krew binary path for integration tests

## Testing Philosophy

- **Unit tests**: Located alongside source files (`*_test.go`)
- **Integration tests**: In `integration_test/` directory, test full workflows
- **Test isolation**: Use `KREW_ROOT` to avoid affecting real installations
- **Race detection**: Always run unit tests with `-race` flag

## Migration Strategy

Krew handles breaking changes through migration code:
- `receiptsmigration`: Handles v0.2.x → v0.3.x receipt format changes
- `indexmigration`: Handles index directory structure changes
- Migrations run automatically in `preRun` hook before any command executes
- Old versions are blocked with helpful error messages if migration isn't complete
