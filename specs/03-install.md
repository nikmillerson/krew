# Krew Install Command Specification

## Command

```bash
kubectl krew install [OPTIONS] [PLUGIN...]
```

## Purpose

Install one or more kubectl plugins from configured indexes or from custom manifests.

## Arguments

- **PLUGIN**: One or more plugin names to install
  - Simple form: `plugin-name` (uses default index)
  - Canonical form: `index-name/plugin-name` (explicit index)
  - Can accept input from stdin (one plugin per line)

## Flags

- `--manifest=FILE`: Install from local manifest file (development only)
- `--manifest-url=URL`: Install from remote manifest URL (development only)
- `--archive=FILE`: Override archive download with local file (requires --manifest or --manifest-url)
- `--no-update-index`: Skip automatic index update before installation (experimental)

## Behavior

### Pre-Installation

**Automatic Index Update:**
- Unless `--no-update-index` or `--manifest` specified
- Runs full index update before installation
- Failure in update doesn't block installation

**Index Check:**
- Ensures at least one index is initialized
- If no index exists and no manifest specified, fails with error

### Input Methods

**1. Positional Arguments:**
```bash
kubectl krew install plugin1 plugin2 plugin3
```

**2. Standard Input:**
```bash
echo "plugin1
plugin2" | kubectl krew install
```
- One plugin name per line
- Empty lines ignored
- Only used if no positional arguments and no manifest flags
- Prints: `Reading plugin names via stdin`

**3. Manifest File:**
```bash
kubectl krew install --manifest=/path/to/plugin.yaml
```

**4. Manifest URL:**
```bash
kubectl krew install --manifest-url=https://example.com/plugin.yaml
```

**Priority:**
- Positional args take precedence over stdin
- If both present, stdin is ignored with warning: `WARNING: Detected stdin, but discarding it because of --manifest or args`

### Plugin Name Validation

For each plugin name:
1. Check for unsafe characters (prevent path traversal)
   - Reject patterns like: `../`, `..\`, directory separators
   - Error: `plugin name "{name}" not allowed`
2. Parse canonical name to extract index and plugin name
3. Validate plugin exists in specified index

### Installation Process

For each plugin:

**1. Pre-Installation Checks:**
- Check if already installed (via receipt file)
- If installed: Skip with warning, continue with next plugin
- Validate plugin name safety

**2. Platform Selection:**
- Load plugin manifest from index
- Match current OS/ARCH against platform selectors
- If no match: Fail with error about platform unavailability
- Select first matching platform entry

**3. Download & Verification:**
- Download archive from platform URI
- Compute SHA256 hash
- Verify against manifest SHA256
- If mismatch: Fail with error

**4. Extraction:**
- Extract to temporary directory
- Support .tar.gz and .zip formats
- Preserve file permissions

**5. File Operations:**
- Execute each file operation from manifest
- Operations: move files from extracted archive to install location
- Format: `from` glob pattern, `to` destination path
- Default `to` is "." (root of plugin directory)

**6. Binary Linking:**
- Identify binary specified in manifest (`bin` field)
- Create symlink from `~/.krew/bin/kubectl-{plugin}` to binary
- On Windows: May use copy instead of symlink

**7. Receipt Generation:**
- Write installation receipt to `~/.krew/receipts/{plugin}.yaml`
- Include: full manifest, installation timestamp, source index
- Receipt format matches `index.Receipt` type

**8. Post-Installation Output:**
```
Installing plugin: plugin-name
Installed plugin: plugin-name
  Use this plugin:
    kubectl plugin-name
  Documentation:
    https://example.com/docs
  Caveats:
   | Any special instructions
   | from the manifest
   /
```

### Security Notices

For plugins from default index only:
- Print security notice after installation
- Message: Warns about auditing and trust
- Not shown for custom index plugins

### Error Handling

**Partial Failure Behavior:**
- Continue installing remaining plugins after individual failures
- Collect failed plugin names
- At end, return error listing all failures
- Successfully installed plugins remain installed

**Already Installed:**
- Log warning: `Skipping plugin "{name}", it is already installed`
- Continue with next plugin
- Not counted as failure

### Archive Override (Development)

When `--archive` specified with manifest:
- Skip downloading from URI
- Use local archive file instead
- Still verify SHA256 if present in manifest
- Useful for testing/development

## Security Requirements for Reimplementation

### Critical: Use Go 1.24 os.Root for All File Operations

**All file operations during installation MUST use Go 1.24's `os.Root` functionality** to prevent:
- Path traversal attacks (Zip Slip vulnerability)
- Symlink escape attacks
- Absolute path exploits
- Directory traversal via `..` in archive entries

See **[12-archive-handling.md](./12-archive-handling.md)** for complete security implementation details.

**Required:**
1. Create `os.Root` for extraction directory
2. Create `os.Root` for installation directory
3. All file reads/writes must go through the respective root
4. Validate all archive entry names before extraction
5. Reject or skip symlinks in archives
6. Never follow symlinks during file operations

**Example secure extraction:**
```go
root := os.NewRoot(installDir)
file, err := root.Create(entryName)  // Cannot escape installDir
```

This prevents vulnerabilities in the current implementation where:
- Archive entries with `../../etc/malicious` can write outside intended directory
- Symlinks in archives can point to sensitive system locations
- File operations don't validate against path traversal

### Archive Extraction Details

For complete details on:
- Supported archive formats (.tar.gz, .zip)
- Extraction mechanics (step-by-step)
- SHA256 verification process
- File operation glob patterns
- Symlink handling
- Error handling at each phase
- Security test requirements

See **[12-archive-handling.md](./12-archive-handling.md)**.

## Output Format

### Standard Error (stderr)

**Progress Messages:**
```
Installing plugin: plugin-name
Installed plugin: plugin-name
  Use this plugin:
    kubectl plugin-name
  Documentation:
    https://example.com/docs
```

**Warnings:**
```
WARNING: Detected stdin, but discarding it because of --manifest or args
Skipping plugin "already-installed", it is already installed
```

**Errors:**
```
failed to install some plugins: [plugin1, plugin2]
plugin "nonexistent" does not exist in the plugin index
```

### Standard Output (stdout)

Silent (no output on success).

## Exit Codes

- **0**: All plugins installed successfully (or all were already installed)
- **Non-zero**: At least one plugin failed to install

## Edge Cases & Error Handling

1. **No Arguments and No Stdin**: Show help message
2. **Plugin Already Installed**: Skip silently, not an error
3. **Platform Not Supported**: Fail with clear message about OS/ARCH
4. **Network Failure During Download**: Fail, don't retry
5. **SHA256 Mismatch**: Fail immediately with security error
6. **Corrupted Archive**: Fail during extraction
7. **Insufficient Permissions**: Fail with permission error
8. **Disk Full**: Fail during file operations
9. **Binary Not Executable**: Installation succeeds but plugin won't run
10. **Manifest File Not Found**: Fail with file not found error
11. **Manifest URL Returns 404**: Fail with HTTP error
12. **Multiple Flags Conflict**: Fail before attempting installation

## Examples

### Install Single Plugin
```bash
kubectl krew install ctx
```

### Install Multiple Plugins
```bash
kubectl krew install ctx ns kubectx
```

### Install from Custom Index
```bash
kubectl krew install myindex/custom-plugin
```

### Install from Explicit Default Index
```bash
kubectl krew install default/krew
```

### Install from Stdin
```bash
kubectl krew list | kubectl krew install
```

### Install from Manifest (Development)
```bash
kubectl krew install --manifest=./plugin.yaml --archive=./plugin.tar.gz
```

### Install Without Index Update
```bash
kubectl krew install --no-update-index plugin-name
```

## Testing Scenarios

### Existing Tests

1. **TestKrewInstall**: Basic installation from index
2. **TestKrewInstallReRun**: Installing already-installed plugin
3. **TestKrewInstallUnsafe**: Reject unsafe plugin names (path traversal)
4. **TestKrewInstall_MultiplePositionalArgs**: Install multiple plugins
5. **TestKrewInstall_Stdin**: Install plugins from stdin
6. **TestKrewInstall_StdinAndPositionalArguments**: Args override stdin
7. **TestKrewInstall_ExplicitDefaultIndex**: Use "default/" prefix
8. **TestKrewInstall_CustomIndex**: Install from custom index
9. **TestKrewInstallNoSecurityWarningForCustomIndex**: No warning for custom indexes
10. **TestKrewInstall_Manifest**: Install from local manifest file
11. **TestKrewInstall_ManifestURL**: Install from remote manifest URL
12. **TestKrewInstall_ManifestAndArchive**: Use local archive with manifest
13. **TestKrewInstall_OnlyArchive**: Fail when only --archive specified
14. **TestKrewInstall_ManifestArgsAreMutuallyExclusive**: --manifest and --manifest-url conflict
15. **TestKrewInstall_NoManifestArgsWhenPositionalArgsSpecified**: Can't mix manifest flags with plugin names

### Additional Test Scenarios

1. **Install nonexistent plugin**: Should fail with clear error
2. **Install with no network**: Should fail during download
3. **Install with SHA256 mismatch**: Should fail with security error
4. **Install with corrupted archive**: Should fail during extraction
5. **Install with malformed manifest**: Should fail during manifest parsing
6. **Install with missing binary in archive**: Should fail during linking
7. **Install to full disk**: Should fail with disk space error
8. **Install with insufficient permissions**: Should fail appropriately
9. **Install same plugin from two indexes**: Both should work, receipts differ
10. **Install with very long plugin name**: Should handle gracefully
11. **Install with manifest missing platform entries**: Should fail with no platform match
12. **Install with manifest having only other OS platforms**: Should fail with platform error
13. **Concurrent installs of same plugin**: Should handle race condition
14. **Install after partial failed install**: Should clean up and retry
15. **Install with symlink already exists but broken**: Should replace symlink
16. **Install with file operations that fail**: Should roll back properly
17. **Install from URL with redirect**: Should follow redirect
18. **Install from URL with slow server**: Should timeout appropriately
19. **Install with manifest using complex file operations**: All operations execute
20. **Install detached plugin then check receipt**: Receipt should show "detached" index
21. **Install empty plugin list from stdin**: Should show help
22. **Install with mixed success/failure**: Successful ones stay installed
23. **Verify receipt timestamps**: creationTimestamp should be set
24. **Verify executable permissions**: Binary should be executable after install
25. **Install with caveats in manifest**: Caveats displayed correctly formatted

## Implementation Notes

- Uses `installation.Install()` as main installation function
- Platform matching uses Kubernetes label selectors (from `k8s.io/apimachinery`)
- Download/verification via `internal/download` package
- SHA256 verification mandatory for security
- File operations support glob patterns in `from` field
- Receipt format defined in `pkg/index.Receipt`
- Index name "detached" reserved for manifest-based installs
- Symlink creation handled per-platform (Windows vs Unix)

## Related Commands

- `kubectl krew search`: Find plugins to install
- `kubectl krew info PLUGIN`: View plugin details before installing
- `kubectl krew upgrade`: Upgrade installed plugins
- `kubectl krew uninstall`: Remove installed plugins
