# Krew Info Command Specification

## Command

```bash
kubectl krew info PLUGIN
```

## Purpose

Display detailed information about a plugin available in a krew index, including metadata, download information, and installation caveats.

## Arguments

- **PLUGIN** (required): Single plugin name
  - Simple form: `plugin-name` (uses default index)
  - Canonical form: `index-name/plugin-name` (explicit index)

## Flags

None.

## Behavior

### Plugin Resolution

1. Parse plugin name to extract index and plugin name:
   - `ctx` → index: "default", plugin: "ctx"
   - `myindex/plugin` → index: "myindex", plugin: "plugin"
2. Load plugin manifest from index at: `~/.krew/index/{index}/plugins/{plugin}.yaml`
3. If not found: Error `plugin "{arg}" not found in index "{index}"`

### Information Displayed

**Always Shown:**
- **NAME**: Plugin name (without index prefix)
- **INDEX**: Index name where plugin was found

**Conditionally Shown:**
- **URI**: Download URL (only if matching platform found and URI present)
- **SHA256**: Checksum (only if matching platform found and SHA256 present)
- **VERSION**: Plugin version (if specified in manifest)
- **HOMEPAGE**: Homepage URL (if specified in manifest)
- **DESCRIPTION**: Long description (if specified in manifest)
- **CAVEATS**: Installation warnings/notes (if specified in manifest, formatted with indent)

### Platform Matching

- Checks manifest platforms against current OS/ARCH
- Uses first matching platform entry
- If no match: URI and SHA256 not shown (still shows other fields)
- Platform selection uses Kubernetes label selectors

### Output Formatting

**Caveats Formatting:**
Uses special indentation wrapper:
```
CAVEATS:
\
 | Line 1 of caveats
 | Line 2 of caveats
 | Additional notes
/
```

**Description:**
Shown as-is, newlines preserved:
```
DESCRIPTION:
This is a plugin that does something useful.
It has multiple lines of description.
```

## Output Format

### Standard Output (stdout)

Example with all fields:
```
NAME: access-matrix
INDEX: default
URI: https://github.com/example/plugin/releases/download/v0.5.0/plugin-linux-amd64.tar.gz
SHA256: abc123def456...
VERSION: v0.5.0
HOMEPAGE: https://github.com/example/plugin
DESCRIPTION:
Show an RBAC access matrix for server resources

This plugin displays a matrix showing which
roles have access to which resources.
CAVEATS:
\
 | This plugin requires kubectl >= 1.20
 | For best results, use with cluster-admin role
/
```

Example with minimal fields (no platform match):
```
NAME: windows-only-plugin
INDEX: default
VERSION: v1.0.0
DESCRIPTION:
A plugin only available on Windows
```

### Error Output (stderr)

```
plugin "nonexistent" not found in index "default"
failed to load plugin manifest
```

## Exit Codes

- **0**: Plugin found and info displayed
- **Non-zero**: Plugin not found or error loading manifest

## Edge Cases & Error Handling

1. **Plugin Doesn't Exist**: Clear error with index name
2. **Manifest Corrupted**: Error during parsing
3. **No Matching Platform**: Shows info but omits URI/SHA256
4. **Missing Optional Fields**: Only shows present fields
5. **Empty Description**: Field omitted
6. **Empty Caveats**: Field omitted
7. **Very Long Description**: Displayed fully, no truncation
8. **Unicode in Description/Caveats**: Displayed as-is
9. **Manifest With Multiple Platforms**: Only shows first matching one

## Examples

### Info for Default Index Plugin
```bash
kubectl krew info ctx
```

### Info for Custom Index Plugin
```bash
kubectl krew info myindex/custom-tool
```

### Info for Plugin Not Available on Current Platform
```bash
kubectl krew info windows-plugin
# (on Linux) Shows info but no URI/SHA256
```

## Testing Scenarios

### Recommended Test Scenarios

1. **Info for existing plugin**: Shows all available fields
2. **Info for nonexistent plugin**: Clear error message
3. **Info for plugin from custom index**: Uses correct index
4. **Info with canonical name**: Explicit index works
5. **Info without platform match**: Omits URI/SHA256
6. **Info for plugin with caveats**: Caveats formatted correctly
7. **Info for plugin without caveats**: Caveats field omitted
8. **Info for plugin with long description**: Full description shown
9. **Info for plugin with homepage**: Homepage displayed
10. **Info for plugin without homepage**: Homepage field omitted
11. **Info for plugin with version**: Version displayed
12. **Info for plugin without version**: Version field omitted
13. **Info for plugin with multiple platforms**: Shows current platform only
14. **Info for detached plugin**: N/A (detached plugins not in index)
15. **Info with unicode characters**: Displays correctly
16. **Info with very long URI**: Full URI shown
17. **Info output to file**: Correctly redirected
18. **Info for plugin with empty description**: Description field omitted
19. **Info when index not initialized**: Appropriate error
20. **Verify caveat indentation**: Pipe characters align correctly
21. **Info for same plugin in multiple indexes**: Each shows separately
22. **Info preserves newlines in description**: Multi-line descriptions correct
23. **Info with manifest having only selector, no URI**: Shows without URI
24. **Info with special characters in caveats**: Displayed correctly
25. **Info called multiple times**: Consistent output

## Implementation Notes

- Uses `indexscanner.LoadPluginByName()` to load manifest
- Path construction via `paths.IndexPluginsPath(index)`
- Plugin name parsing via `pathutil.CanonicalPluginName()`
- Platform matching via `installation.GetMatchingPlatform()`
- Caveat formatting via `indent()` helper function
- Output directly to stdout using `fmt.Fprintf()`
- Field presence checks before printing optional fields
- No caching of manifest data (loads fresh each time)

## Related Commands

- `kubectl krew search`: Find plugins before getting info
- `kubectl krew install`: Install plugin after viewing info
- `kubectl krew list`: See if plugin already installed
