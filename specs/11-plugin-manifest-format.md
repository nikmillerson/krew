# Plugin Manifest Format Specification

## Purpose

The plugin manifest is a YAML file that describes how to install a kubectl plugin. It contains metadata, download URLs, platform-specific instructions, and installation notes.

## File Location

- In index repository: `plugins/{plugin-name}.yaml`
- Local filesystem: Any path when using `--manifest` flag
- Remote URL: Any HTTP(S) URL when using `--manifest-url` flag

## File Structure

### Complete Example

```yaml
apiVersion: krew.googlecontainertools.github.com/v1alpha2
kind: Plugin
metadata:
  name: ctx
spec:
  version: v0.9.4
  shortDescription: Switch between kubectl contexts
  description: |
    This plugin allows you to quickly switch between different
    kubectl contexts. It is particularly useful when working with
    multiple Kubernetes clusters.

    Run 'kubectl ctx' to see all contexts.
    Run 'kubectl ctx <name>' to switch context.
  caveats: |
    This plugin requires kubectl version 1.12 or higher.

    For best experience, ensure your kubeconfig is properly configured.
  homepage: https://github.com/ahmetb/kubectx
  platforms:
  - selector:
      matchLabels:
        os: linux
        arch: amd64
    uri: https://github.com/ahmetb/kubectx/releases/download/v0.9.4/kubectx_v0.9.4_linux_x86_64.tar.gz
    sha256: abc123def456...
    files:
    - from: kubectx
      to: .
    - from: LICENSE
      to: .
    bin: kubectx
  - selector:
      matchLabels:
        os: linux
        arch: arm64
    uri: https://github.com/ahmetb/kubectx/releases/download/v0.9.4/kubectx_v0.9.4_linux_arm64.tar.gz
    sha256: def456ghi789...
    files:
    - from: kubectx
      to: .
    bin: kubectx
  - selector:
      matchLabels:
        os: darwin
        arch: amd64
    uri: https://github.com/ahmetb/kubectx/releases/download/v0.9.4/kubectx_v0.9.4_darwin_x86_64.tar.gz
    sha256: ghi789jkl012...
    files:
    - from: kubectx
      to: .
    bin: kubectx
  - selector:
      matchLabels:
        os: darwin
        arch: arm64
    uri: https://github.com/ahmetb/kubectx/releases/download/v0.9.4/kubectx_v0.9.4_darwin_arm64.tar.gz
    sha256: jkl012mno345...
    files:
    - from: kubectx
      to: .
    bin: kubectx
  - selector:
      matchLabels:
        os: windows
        arch: amd64
    uri: https://github.com/ahmetb/kubectx/releases/download/v0.9.4/kubectx_v0.9.4_windows_x86_64.zip
    sha256: mno345pqr678...
    files:
    - from: kubectx.exe
      to: .
    bin: kubectx.exe
```

## Field Specifications

### Top-Level Fields

**apiVersion** (string, required)
- Current value: `krew.googlecontainertools.github.com/v1alpha2`
- Identifies manifest format version
- Used for compatibility checking
- Value defined in `pkg/constants.CurrentAPIVersion`

**kind** (string, required)
- Must be: `Plugin`
- Identifies resource type
- Value defined in `pkg/constants.PluginKind`

**metadata** (object, required)
- Kubernetes-style metadata object
- Contains plugin identification

**spec** (object, required)
- Plugin specification
- Contains all installation details

### Metadata Fields

**metadata.name** (string, required)
- Plugin name (without index prefix)
- Must match filename (without `.yaml` extension)
- Must be safe (validated by `validation.IsSafePluginName()`)
- Used as: `kubectl {name}`
- Examples: `ctx`, `ns`, `access-matrix`

### Spec Fields

**spec.version** (string, optional but recommended)
- Plugin version string
- Should follow semantic versioning
- Used for upgrade detection
- Empty string allowed (treated as no version)
- Examples: `v0.9.4`, `1.2.3`, `v1.0.0-beta.1`

**spec.shortDescription** (string, optional)
- Brief one-line description
- Shown in search results
- Truncated to 50 chars in table view
- Plain text (no formatting)
- Example: `Switch between kubectl contexts`

**spec.description** (string, optional)
- Detailed multi-line description
- Shown in info command
- Supports newlines
- Plain text (no formatting)
- Can be multi-paragraph

**spec.homepage** (string, optional)
- URL to plugin homepage/documentation
- Shown in info command
- Shown after installation
- Should be full URL with protocol
- Example: `https://github.com/example/plugin`

**spec.caveats** (string, optional)
- Important installation notes
- Shown after installation
- Formatted with special indentation
- Can be multi-line
- Examples: Requirements, warnings, tips
- Format:
  ```
  \
   | Line 1 of caveats
   | Line 2 of caveats
  /
  ```

**spec.platforms** (array, required)
- List of platform-specific installation instructions
- At least one platform entry required
- Processed in order until match found
- Each entry is a platform object (see below)

### Platform Object

Each platform entry supports a specific OS/architecture combination.

**platform.selector** (object, required)
- Kubernetes-style label selector
- Specifies which OS/ARCH this entry supports
- Uses `k8s.io/apimachinery/pkg/apis/meta/v1.LabelSelector`

**platform.selector.matchLabels** (object, optional)
- Key-value pairs for matching
- Common keys: `os`, `arch`
- All labels must match for platform to be selected
- Example:
  ```yaml
  matchLabels:
    os: linux
    arch: amd64
  ```

**platform.selector.matchExpressions** (array, optional)
- Advanced matching with operators
- Operators: `In`, `NotIn`, `Exists`, `DoesNotExist`
- Example:
  ```yaml
  matchExpressions:
  - key: os
    operator: In
    values: [linux, darwin]
  - key: arch
    operator: NotIn
    values: [arm]
  ```

**platform.uri** (string, required for most cases)
- Download URL for plugin archive
- Supports `.tar.gz` and `.zip` formats
- Should use HTTPS for security
- Can be any HTTP(S) URL
- Example: `https://github.com/user/repo/releases/download/v1.0/plugin.tar.gz`

**platform.sha256** (string, required for most cases)
- SHA256 checksum of downloaded archive
- Hexadecimal string (64 characters)
- Used for download verification
- Mandatory for security
- Generated with: `shasum -a 256 file.tar.gz`

**platform.files** (array, required)
- List of file operations
- Executed after extraction
- Each operation moves/copies files
- Similar to `mv` command behavior

**platform.files[].from** (string, required)
- Source path/pattern in extracted archive
- Supports glob patterns (`*`, `**`, `?`)
- Relative to archive root
- Examples: `plugin-binary`, `bin/*`, `**/*.so`

**platform.files[].to** (string, required)
- Destination path in installation directory
- Relative to plugin installation root
- Default: `.` (root directory)
- Creates intermediate directories as needed
- Examples: `.`, `bin/`, `lib/`

**platform.bin** (string, required)
- Path to main plugin executable
- Relative to installation directory (after file operations)
- Will be symlinked as `kubectl-{name}` in bin directory
- Must be executable in archive
- Examples: `plugin-name`, `bin/plugin`, `plugin.exe`

## Label Values

### Operating Systems (os)

Standard values:
- `linux`: Linux systems
- `darwin`: macOS systems
- `windows`: Windows systems

Detected from Go's `runtime.GOOS`.

### Architectures (arch)

Standard values:
- `amd64`: 64-bit x86 (most common)
- `386`: 32-bit x86
- `arm64`: 64-bit ARM (Apple Silicon, some Linux)
- `arm`: 32-bit ARM

Detected from Go's `runtime.GOARCH`.

## File Operations Behavior

### Glob Patterns

The `from` field supports:
- `*`: Match any characters except `/`
- `**`: Match any characters including `/` (recursive)
- `?`: Match single character
- `[abc]`: Match character class
- `{a,b}`: Match alternatives

### Operation Semantics

Similar to Unix `mv` command:
```yaml
files:
- from: source/*        # Moves all files in source/
  to: dest/             # To dest/ directory
- from: file.txt        # Moves single file
  to: .                 # To root of installation
- from: "*.txt"         # Moves all .txt files
  to: docs/             # To docs/ subdirectory
```

### Special Cases

**Default `to` value:**
- If omitted: treated as `.` (root)
- Most plugins use explicit `.` for clarity

**Directory creation:**
- Intermediate directories auto-created
- No need to pre-create destination dirs

**Overwrites:**
- Files with same name overwritten
- No warning or error

**Archive root:**
- Some archives have top-level directory
- Use `from: archive-name/*` to skip it
- Or `from: "*/actual-files"` for glob

## Archive Format Requirements

**Supported Formats:**
- `.tar.gz` (most common)
- `.zip` (Windows preference)
- No other formats supported

**Archive Contents:**
- Should contain plugin binary
- May contain LICENSE, README, etc.
- Should have proper file permissions (Unix)
- Binary must be executable

**Common Issues:**
- Archive with nested directories: Use glob patterns in `from`
- Missing executable bit: Binary won't run after install
- Absolute paths in archive: May cause issues
- Symlinks in archive: Behavior undefined, avoid

## Validation Rules

### Name Validation
- Must be safe filename
- No path separators (`/`, `\`)
- No parent references (`..`)
- No hidden file prefix (`.`)
- Alphanumeric, hyphens, underscores typically safe

### Version Format
- No strict format enforced
- Semantic versioning recommended
- Used for string comparison if not semver
- Empty string allowed

### URI Validation
- Must be valid HTTP(S) URL
- Should resolve to downloadable file
- Should use HTTPS for security
- File extension should match format

### SHA256 Format
- Must be 64 hexadecimal characters
- Case-insensitive
- No prefix (`0x`, etc.)
- Whitespace trimmed

## Complete Minimal Example

Minimal valid manifest:

```yaml
apiVersion: krew.googlecontainertools.github.com/v1alpha2
kind: Plugin
metadata:
  name: myplugin
spec:
  version: v1.0.0
  platforms:
  - selector:
      matchLabels:
        os: linux
        arch: amd64
    uri: https://example.com/myplugin.tar.gz
    sha256: 0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
    files:
    - from: "*"
      to: "."
    bin: myplugin
```

## Complex Example with Multiple Platforms

```yaml
apiVersion: krew.googlecontainertools.github.com/v1alpha2
kind: Plugin
metadata:
  name: multiplugin
spec:
  version: v2.0.0
  shortDescription: A plugin for multiple platforms
  homepage: https://example.com
  platforms:
  - selector:
      matchExpressions:
      - key: os
        operator: In
        values: [linux, darwin]
      - key: arch
        operator: In
        values: [amd64, arm64]
    uri: https://example.com/multiplugin-unix-{arch}.tar.gz
    sha256: abc123...
    files:
    - from: multiplugin-*/bin/*
      to: .
    - from: multiplugin-*/LICENSE
      to: .
    bin: multiplugin
  - selector:
      matchLabels:
        os: windows
        arch: amd64
    uri: https://example.com/multiplugin-windows.zip
    sha256: def456...
    files:
    - from: multiplugin.exe
      to: .
    bin: multiplugin.exe
```

## Common Patterns

### Single Binary Plugin
```yaml
files:
- from: plugin-name
  to: .
bin: plugin-name
```

### Plugin with Dependencies
```yaml
files:
- from: bin/plugin
  to: .
- from: lib/*
  to: lib/
bin: plugin
```

### Plugin with Wrapper Script
```yaml
files:
- from: plugin.sh
  to: .
- from: plugin-binary
  to: bin/
bin: plugin.sh
```

### Cross-platform with Different Names
```yaml
platforms:
- selector:
    matchLabels:
      os: linux
  files:
  - from: plugin-linux
    to: plugin
  bin: plugin
- selector:
    matchLabels:
      os: windows
  files:
  - from: plugin.exe
    to: .
  bin: plugin.exe
```

## Receipt Format

After installation, a receipt file is created at `~/.krew/receipts/{plugin}.yaml`.

Receipt includes:
- Complete original manifest (embedded)
- Installation metadata
- Source index information
- Creation timestamp

Example receipt:
```yaml
apiVersion: krew.googlecontainertools.github.com/v1alpha2
kind: Plugin
metadata:
  creationTimestamp: "2023-01-15T10:30:00Z"
  name: ctx
spec:
  # ... full manifest ...
status:
  source:
    name: default  # or "detached" for manifest-installed plugins
```

## Implementation Notes

- Manifest parsing: `sigs.k8s.io/yaml` library
- Type definitions: `pkg/index/types.go`
- Validation: `internal/index/validation` package
- Platform matching: `k8s.io/apimachinery/pkg/apis/meta/v1` label selectors
- File operations: `internal/installation/move.go`
- Receipt handling: `internal/installation/receipt` package
