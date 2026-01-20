# Krew Search Command Specification

## Command

```bash
kubectl krew search [KEYWORD]
```

## Purpose

Discover kubectl plugins available in configured indexes. Without arguments, lists all available plugins. With a keyword, performs fuzzy search across plugin names and descriptions.

## Arguments

- **KEYWORD** (optional): Search term for fuzzy matching. Multiple words are concatenated without spaces.

## Behavior

### Prerequisites

- At least one index must be initialized (checked via `checkIndex` pre-run)
- If no index exists, returns error: `krew local plugin index is not initialized (run "kubectl krew update")`

### Search Algorithm

**Without Keyword (List All):**
1. Load all plugins from all configured indexes
2. Display in table format sorted by name
3. Show installation status for each

**With Keyword:**
1. Concatenate all arguments into single search term (no spaces)
2. Perform fuzzy search using `github.com/sahilm/fuzzy` library:
   - **First pass**: Search plugin names
   - **Second pass**: Search short descriptions (only if score > 0 and not already matched)
3. Return results in match order (name matches first, then description matches)

### Platform Availability Detection

For each plugin:
- If installed: Status = `yes`
- If platform matches (OS/arch selector): Status = `no` (available but not installed)
- If no matching platform: Status = `unavailable on {os}/{arch}`

### Output Format

Table with columns:
- **NAME**: Display name (with index prefix for non-default indexes)
- **DESCRIPTION**: Short description (truncated to 50 chars with "..." if longer)
- **INSTALLED**: Installation status

Example:
```
NAME               DESCRIPTION                                         INSTALLED
ctx                Switch between kubectl contexts                    no
default/ctx        Switch between kubectl contexts                    yes
custom-idx/ctx     Switch between kubectl contexts                    no
access-matrix      Show an access matrix for resources                unavailable on linux/arm64
```

### Sorting

- Results sorted alphabetically by display name
- Cross-index: `custom/plugin` comes before `default/plugin` (lexicographic)

### Multi-Index Handling

When multiple indexes have the same plugin:
- All versions listed separately
- Distinguished by index name prefix
- Each shows independent installation status
- Example: Both `ctx` and `foo/ctx` can appear if plugin exists in both indexes

## Output Format

### Standard Output (stdout)

Table format using `text/tabwriter` with 2-space padding:
```
NAME         DESCRIPTION                     INSTALLED
plugin1      Does something useful           yes
plugin2      Another helpful tool            no
idx2/plugin3 Tool from custom index          no
```

### No Results

If search returns no matches, outputs nothing (empty result).

### Error Output (stderr)

Errors go to stderr, not affecting table output.

## Exit Codes

- **0**: Success (even if no results found)
- **Non-zero**: Error loading indexes or plugins

## Edge Cases & Error Handling

1. **Empty Index**: If index has no plugins, shows empty list
2. **Parse Errors in Manifests**:
   - Logged at V(1) level
   - Plugin skipped
   - Other plugins still shown
3. **Multiple Word Search**: "access matrix" becomes "accessmatrix" (no space)
4. **Case Sensitivity**: Fuzzy search is case-insensitive
5. **Special Characters in Search**: Handled by fuzzy search algorithm
6. **Very Long Descriptions**: Truncated to 50 chars with "..." suffix
7. **Plugin in Multiple Indexes with Same Name**:
   - Only one from default index shows without prefix
   - Others show with index/ prefix
8. **Platform Selectors**:
   - Uses Kubernetes label selector matching
   - Checks current OS/ARCH against manifest selectors

## Examples

### List All Plugins
```bash
kubectl krew search
```

Shows all plugins from all indexes.

### Search by Name
```bash
kubectl krew search krew
```

Shows plugins with "krew" in name (fuzzy match).

### Search by Description
```bash
kubectl krew search "context switch"
```

Shows plugins with "contextswitch" in name or description.

### Search with Custom Index
```bash
kubectl krew search access
```

May return `access-matrix`, `custom/access-tool`, etc.

## Testing Scenarios

### Existing Tests

1. **TestKrewSearchAll**: Lists all plugins from default index
2. **TestKrewSearchOne**: Search for specific plugin ("krew")
3. **TestKrewSearchMultiIndex**: Show plugins from multiple indexes with correct prefixes
4. **TestKrewSearchMultiIndexSortedByDisplayName**: Verify alphabetical sorting

### Additional Test Scenarios

1. **Search with no matches**: Should return empty output (not error)
2. **Search with partial match**: Fuzzy search should find "krw" matching "krew"
3. **Search with exact match**: Should rank exact matches higher
4. **Search in descriptions only**: Plugin name doesn't match but description does
5. **Search with special characters**: "access-matrix" with dash
6. **Search with case variations**: "KREW", "Krew", "krew" all match
7. **Search with multiple words**: "access matrix" (space removed)
8. **Search with unicode characters**: Non-ASCII search terms
9. **List plugins when some have long descriptions**: Truncation works correctly
10. **List plugins with unavailable platforms**: Shows "unavailable on..."
11. **Search when index load fails**: Graceful error handling
12. **Search with installed plugins**: Shows "yes" in correct column
13. **Search with mixed installed/not installed**: Correct status for each
14. **Search across 3+ indexes**: All plugins shown with correct prefixes
15. **Search when plugin exists in both default and custom index**: Both shown
16. **Search with only custom indexes (no default)**: Works correctly
17. **Verify column alignment**: Tab-separated columns align properly
18. **Search with empty keyword**: Should list all (same as no argument)
19. **Description matching scored correctly**: Relevant results appear
20. **Deduplication works**: Plugin found in both name and description only listed once

## Implementation Notes

- Uses `indexscanner.LoadPluginListFromFS()` to load plugins
- Fuzzy matching via `github.com/sahilm/fuzzy` library
- Table rendering via `text/tabwriter.NewWriter()` with 0 width, 0 pad, 2 space, ' ' char
- Platform detection uses `installation.GetMatchingPlatform()`
- Installed plugin check uses receipt files from `InstallReceiptsPath()`
- Description truncation considers character count, not byte count

## Related Commands

- `kubectl krew info PLUGIN`: Get detailed information about a plugin
- `kubectl krew install PLUGIN`: Install a plugin found via search
- `kubectl krew list`: List only installed plugins
