# Krew List Command Specification

## Command

```bash
kubectl krew list
```

Aliases: `ls`

## Purpose

Display a list of installed kubectl plugins managed by krew, showing their names and versions.

## Arguments

None.

## Flags

None.

## Behavior

### Plugin Discovery

1. Read all receipt files from `~/.krew/receipts/`
2. Parse each receipt to extract:
   - Plugin name
   - Installed version
   - Source index name
3. Collect all successfully parsed receipts

### Output Determination

**Terminal Output (TTY):**
When stdout is a terminal:
- Display formatted table with columns: `PLUGIN` and `VERSION`
- Use tabwriter for aligned columns (2 space padding)
- Sort alphabetically by plugin display name
- Include index prefix for non-default indexes

**Piped/Redirected Output (Non-TTY):**
When stdout is piped or redirected:
- Output only plugin names (one per line)
- Newline-separated list
- Sorted alphabetically
- Include index prefix for non-default indexes
- No headers, no version info
- Suitable for piping to `kubectl krew install`

### Display Names

- Default index plugins: Show simple name (e.g., `ctx`)
- Custom index plugins: Show with prefix (e.g., `myindex/plugin`)
- Format matches search command display names

### Sorting

- Alphabetical sort by display name
- Case-sensitive sort (uppercase before lowercase in ASCII)
- Cross-index: `custom/plugin` sorted alongside `default-plugin`

## Output Format

### Terminal Output (TTY)

Table format:
```
PLUGIN       VERSION
ctx          v0.9.4
myindex/foo  v1.2.0
ns           v0.9.4
```

### Piped Output (Non-TTY)

Plain list:
```
ctx
myindex/foo
ns
```

### Empty List

If no plugins installed:
- Terminal: Empty table (only headers shown)
- Piped: Empty output (no lines)

## Exit Codes

- **0**: Success (even if no plugins installed)
- **Non-zero**: Error reading receipts directory or parsing receipts

## Edge Cases & Error Handling

1. **No Plugins Installed**: Shows empty list (not an error)
2. **Receipts Directory Missing**: Error (should exist if krew initialized)
3. **Corrupted Receipt Files**: Logged, skipped, continues with others
4. **Receipt Without Version**: Shows empty version (or skips)
5. **Receipt Without Index**: Treated as default index
6. **Multiple Receipt Files with Same Name**: Undefined (shouldn't happen)
7. **Receipt Name Doesn't Match Content**: Uses receipt filename
8. **Very Long Plugin Names**: May affect table formatting
9. **Very Long Versions**: May affect table formatting

## Examples

### List Installed Plugins (Terminal)
```bash
kubectl krew list
```
Output:
```
PLUGIN    VERSION
ctx       v0.9.4
krew      v0.4.4
ns        v0.9.4
```

### List for Piping
```bash
kubectl krew list > installed.txt
cat installed.txt
```
Output:
```
ctx
krew
ns
```

### Backup and Restore Pattern
```bash
# Backup
kubectl krew list > backup.txt

# Later restore
kubectl krew install < backup.txt
```

### Using Alias
```bash
kubectl krew ls
```

## Testing Scenarios

### Existing Tests

Based on `list_test.go` analysis (not fully reviewed):
- Likely tests for basic list functionality
- Terminal vs piped output detection
- Sorting verification

### Recommended Test Scenarios

1. **List with no plugins installed**: Empty output
2. **List with single plugin**: Shows correctly
3. **List with multiple plugins**: All shown, sorted
4. **List output when piped**: Plain name list (no table)
5. **List output to terminal**: Table format with headers
6. **List with custom index plugins**: Shows index prefix
7. **List with mix of default and custom**: Correct prefixes
8. **List sorted alphabetically**: Verify sort order
9. **List with detached plugins**: Shows "detached" prefix or special handling
10. **Pipe list to install command**: Round-trip works
11. **List after fresh install**: New plugin appears
12. **List after uninstall**: Plugin removed from list
13. **List after upgrade**: Same plugin, updated version
14. **List with corrupted receipt**: Skips and continues
15. **List with receipt but no binary**: Still shows in list
16. **List with broken symlink**: Still shows in list
17. **Verify table column alignment**: Columns properly aligned
18. **List with very long plugin name**: Formatting works
19. **List with very long version**: Formatting works
20. **List stdout vs stderr**: Output goes to correct stream
21. **List in non-English locale**: Works correctly
22. **List with unicode plugin names**: If supported, displays correctly
23. **Concurrent list operations**: No corruption
24. **List when receipts directory is empty**: Shows empty list
25. **List using ls alias**: Identical to list command

## Implementation Notes

- Uses `installation.GetInstalledPluginReceipts()` to load receipts
- Terminal detection via `isatty.IsTerminal()` and `isatty.IsCygwinTerminal()`
- Display name formatting via `displayName()` helper (same as search)
- Index name extraction via `indexOf()` helper (reads receipt source)
- Table rendering via `text/tabwriter.NewWriter()` with 0 width, 0 pad, 2 space
- Sorting via `sort.Strings()` for piped output, `sortByFirstColumn()` for table
- Receipt path constructed via `environment.Paths.InstallReceiptsPath()`

## Related Commands

- `kubectl krew install`: Install plugins to appear in list
- `kubectl krew uninstall`: Remove plugins from list
- `kubectl krew upgrade`: Update versions shown in list
- `kubectl krew search`: Find plugins not in list
- `kubectl plugin list`: Show ALL kubectl plugins (krew and non-krew)
