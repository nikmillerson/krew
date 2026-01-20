# Krew Quick Reference

## All Commands

```bash
kubectl krew update                      # Update plugin indexes
kubectl krew search [KEYWORD]            # Search for plugins
kubectl krew install PLUGIN...           # Install plugins
kubectl krew upgrade [PLUGIN...]         # Upgrade plugins
kubectl krew uninstall PLUGIN...         # Uninstall plugins
kubectl krew list                        # List installed plugins
kubectl krew info PLUGIN                 # Show plugin details
kubectl krew version                     # Show krew version
kubectl krew index list                  # List plugin indexes
kubectl krew index add NAME URL          # Add custom index
kubectl krew index remove NAME           # Remove index
```

## Command Aliases

```bash
kubectl krew ls          # Alias for: kubectl krew list
kubectl krew remove      # Alias for: kubectl krew uninstall
kubectl krew rm          # Alias for: kubectl krew uninstall
```

## Common Flags

```bash
--manifest=FILE          # Install from local manifest (install)
--manifest-url=URL       # Install from remote manifest (install)
--archive=FILE           # Use local archive file (install, with manifest)
--no-update-index        # Skip index update (install, upgrade)
--force                  # Force removal despite installed plugins (index remove)
-v=LEVEL                 # Verbosity level (0-4, all commands)
```

## Plugin Name Formats

```bash
plugin-name              # Simple name (uses default index)
default/plugin-name      # Explicit default index
custom-idx/plugin-name   # Custom index
```

## Directory Structure

```
~/.krew/
├── bin/                    # Symlinks (add to $PATH)
│   └── kubectl-*           # Plugin executables
├── index/                  # Plugin indexes (git repos)
│   ├── default/            # Default krew index
│   │   └── plugins/        # Plugin manifests (.yaml)
│   └── custom/             # Custom indexes
├── store/                  # Installed plugins
│   └── {plugin}/           # Plugin directories
│       └── {version}/      # Versioned installations
└── receipts/               # Installation receipts
    └── {plugin}.yaml       # Installation metadata
```

## Environment Variables

```bash
KREW_ROOT                # Override ~/.krew directory
KREW_NO_UPGRADE_CHECK    # Disable upgrade notifications
KREW_BINARY              # Path to krew binary (testing)
KREW_OS                  # Override OS detection (testing)
KREW_ARCH                # Override arch detection (testing)
```

## Typical Workflows

### First Time Setup
```bash
# Install krew first (see https://krew.sigs.k8s.io)
kubectl krew update                     # Initialize index
kubectl krew search                     # Browse available plugins
kubectl krew install ctx ns             # Install some plugins
```

### Regular Usage
```bash
kubectl krew search kubectl             # Find plugins
kubectl krew info ctx                   # View plugin details
kubectl krew install ctx                # Install plugin
kubectl ctx                             # Use plugin (note: no 'krew')
kubectl krew list                       # See installed plugins
kubectl krew upgrade                    # Update all plugins
kubectl krew upgrade ctx                # Update specific plugin
kubectl krew uninstall ctx              # Remove plugin
```

### Custom Index
```bash
kubectl krew index add mycompany https://github.com/mycompany/krew-index.git
kubectl krew search                     # Shows plugins from all indexes
kubectl krew install mycompany/plugin   # Install from custom index
kubectl krew index list                 # View all indexes
kubectl krew index remove mycompany     # Remove custom index
```

### Backup & Restore
```bash
# Backup installed plugins
kubectl krew list > installed-plugins.txt

# Restore on new system
kubectl krew update
kubectl krew install < installed-plugins.txt
```

### Development
```bash
# Install from local manifest
kubectl krew install --manifest=./plugin.yaml

# Install with local archive (no download)
kubectl krew install --manifest=./plugin.yaml --archive=./plugin.tar.gz

# Install from URL
kubectl krew install --manifest-url=https://example.com/plugin.yaml

# Skip index update (faster)
kubectl krew install --no-update-index plugin-name
```

## Exit Codes

```
0          Success
Non-zero   Error (specific code not guaranteed)
```

## Output Streams

```
stdout     Command results (lists, tables, info)
stderr     Progress messages, warnings, errors
```

## Common Error Messages

```
"plugin not found in index"
→ Run: kubectl krew update

"Index not initialized"
→ Run: kubectl krew update

"No matching platform"
→ Plugin not available for your OS/ARCH

"Already installed"
→ Plugin already installed (not an error)
→ Use: kubectl krew upgrade PLUGIN

"Permission denied"
→ Check ~/.krew directory permissions

"Failed to update index"
→ Check network connection
→ May need to: rm -rf ~/.krew/index/NAME && kubectl krew index add NAME URL
```

## Platform Identifiers

### Operating Systems
```
linux      Linux
darwin     macOS
windows    Windows
```

### Architectures
```
amd64      64-bit x86 (Intel/AMD)
arm64      64-bit ARM (Apple Silicon, modern ARM)
386        32-bit x86
arm        32-bit ARM
```

## Manifest YAML (Quick)

```yaml
apiVersion: krew.googlecontainertools.github.com/v1alpha2
kind: Plugin
metadata:
  name: plugin-name
spec:
  version: v1.0.0
  shortDescription: One-line description
  description: |
    Longer description
    with multiple lines
  homepage: https://example.com
  caveats: |
    Installation notes
    and warnings
  platforms:
  - selector:
      matchLabels:
        os: linux
        arch: amd64
    uri: https://example.com/plugin-linux-amd64.tar.gz
    sha256: abc123...
    files:
    - from: "*"
      to: "."
    bin: plugin-name
```

## Logging Verbosity

```bash
-v=0    # Default (errors, warnings)
-v=1    # Info messages
-v=2    # Detailed operations
-v=3    # Debug flows
-v=4    # Very detailed (file ops, git commands)
```

Example:
```bash
kubectl krew -v=4 install plugin-name
```

## Security

- **Download Verification**: SHA256 mandatory
- **Plugin Execution**: By kubectl, not krew
- **Custom Indexes**: Show security warning
- **Path Validation**: Prevents traversal attacks
- **No Code Execution**: Krew doesn't run plugin code

### Security for Reimplementation

**CRITICAL:** Any reimplementation must use **Go 1.24's os.Root** functionality for all file operations to prevent:
- Path traversal (Zip Slip) attacks
- Symlink escape vulnerabilities
- Absolute path exploits

See **specs/12-archive-handling.md** for complete security requirements.

## Performance Tips

- Index update is network-bound (uses git)
- Plugin install dominated by download time
- Search scans all manifests (no caching)
- Use `--no-update-index` to skip updates

## Getting Help

```bash
kubectl krew --help              # General help
kubectl krew COMMAND --help      # Command-specific help
kubectl krew version             # Version and diagnostics
```

## Online Resources

- Documentation: https://krew.sigs.k8s.io
- Index Repository: https://github.com/kubernetes-sigs/krew-index
- Issue Tracker: https://github.com/kubernetes-sigs/krew/issues
- Plugin List: https://krew.sigs.k8s.io/plugins/

## Plugin Development

To create a plugin:
1. Write manifest YAML
2. Host binary releases (GitHub releases recommended)
3. Calculate SHA256 checksums
4. Test with `kubectl krew install --manifest=./plugin.yaml`
5. Submit to krew-index repository

See: https://krew.sigs.k8s.io/docs/developer-guide/

## Important Behaviors

- **Idempotent**: Re-running commands is safe
- **Partial Failure**: Install continues after individual failures
- **No Locking**: Concurrent operations may conflict
- **Last Write Wins**: No transaction handling
- **PATH Check**: Warns if ~/.krew/bin not in $PATH
- **Auto Migrate**: Handles version upgrades automatically
- **Random Check**: Upgrade check ~40% of runs (avoid rate limits)
