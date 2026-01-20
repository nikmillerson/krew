# Krew Engineering Specifications

This directory contains comprehensive engineering specifications for recreating the krew plugin manager from scratch. Each specification documents observable behavior, implementation requirements, and test scenarios.

## Purpose

These specifications serve as:
1. **Design Documentation**: Complete functional requirements for each component
2. **Test Specification**: Comprehensive test scenarios (existing and recommended)
3. **Recreation Blueprint**: Detailed enough to reimplement the system
4. **API Contract**: Defines user-observable behavior precisely

## Document Organization

### 00-overview.md
**System Architecture Overview**
- Core directory structure and design
- Plugin index system architecture
- Installation flow and lifecycle
- Bootstrap and self-installation
- Environment variables and configuration
- Observable system states

Read this first to understand the overall system design.

### 01-update.md
**`kubectl krew update` Command**
- Index synchronization via git
- Automatic index initialization
- New plugins and upgrade notifications
- Multi-index update handling
- Implicit invocation by other commands

### 02-search.md
**`kubectl krew search` Command**
- Plugin discovery and listing
- Fuzzy search algorithm (name and description)
- Platform availability detection
- Multi-index display with prefixes
- Table formatting and sorting

### 03-install.md
**`kubectl krew install` Command**
- Plugin installation from indexes or manifests
- Multiple input methods (args, stdin, manifest)
- Platform selection and matching
- Download, verification, and extraction
- Binary linking and receipt generation
- Partial failure handling

### 04-upgrade.md
**`kubectl krew upgrade` Command**
- Plugin upgrade to newer versions
- All vs. specific plugin upgrade modes
- Version comparison using semver
- Detached plugin handling
- Platform mismatch scenarios

### 05-uninstall.md
**`kubectl krew uninstall` Command**
- Plugin removal process
- Symlink and file cleanup
- Receipt deletion
- Immediate failure behavior

### 06-list.md
**`kubectl krew list` Command**
- Installed plugin enumeration
- Terminal vs. piped output modes
- Display name formatting
- Receipt-based plugin discovery

### 07-info.md
**`kubectl krew info` Command**
- Detailed plugin information display
- Manifest field presentation
- Platform-specific URI/SHA256 display
- Caveat and description formatting

### 08-version.md
**`kubectl krew version` Command**
- Build information display
- Directory path diagnostics
- Platform detection information
- Configuration verification

### 09-index.md
**Index Management Commands**
- `kubectl krew index list`: View configured indexes
- `kubectl krew index add`: Add custom plugin sources
- `kubectl krew index remove`: Remove plugin indexes
- Safety checks and warnings

### 10-system-behavior.md
**Cross-Cutting System Behaviors**
- PATH checking and warnings
- Directory initialization
- Migration system (receipts, index)
- Upgrade notifications
- Error handling philosophy
- Logging and verbosity
- Platform detection
- Naming conventions
- Concurrency and safety
- Security considerations
- Terminal detection
- Environment variables

### 11-plugin-manifest-format.md
**Plugin Manifest YAML Specification**
- Complete manifest structure
- Field-by-field documentation
- Platform selectors (Kubernetes label selectors)
- File operations and glob patterns
- Archive format requirements
- Validation rules
- Receipt format
- Common patterns and examples

### 12-archive-handling.md
**Archive Extraction and File Operations**
- Current implementation mechanics (download, extraction, file operations)
- Supported archive formats (.tar.gz, .zip)
- Detailed extraction process step-by-step
- Security vulnerabilities in current implementation
- **Critical: Go 1.24 os.Root security requirements**
- Path traversal and symlink attack prevention
- Secure reimplementation guidance
- Complete security test requirements
- Implementation checklist for secure file operations

**Read this for security-critical implementation details.**

## How to Use These Specs

### For Implementation

1. **Start with 00-overview.md** to understand the system architecture
2. **Read 11-plugin-manifest-format.md** to understand the data format
3. **Read 10-system-behavior.md** for cross-cutting concerns
4. **Implement commands** in order of dependency:
   - Start with `update` (foundational)
   - Then `search` and `info` (read-only)
   - Then `install` (complex, builds on update)
   - Then `upgrade` (builds on install)
   - Then `uninstall`, `list`, `version` (simpler)
   - Finally `index` commands (builds on all)

### For Testing

Each spec includes:
- **Existing Tests**: What's currently tested in integration tests
- **Additional Test Scenarios**: Comprehensive list of recommended tests

Use these to:
1. Verify existing implementation matches specs
2. Identify gaps in test coverage
3. Create comprehensive test suites
4. Validate reimplementation

### For Bug Fixes

1. Find the relevant command spec
2. Locate the behavior description
3. Check edge cases and error handling sections
4. Verify test scenarios cover the bug
5. Add new test scenario if needed

### For API Changes

1. Update relevant spec file first
2. Document new behavior precisely
3. Add edge cases and error conditions
4. Define test scenarios
5. Update implementation to match
6. Verify tests pass

## Specification Conventions

### Observable Behavior Focus

Specifications describe:
- What the user sees (output, exit codes)
- What the system does (files created, network calls)
- Error messages and warnings
- Interactive behavior

Specifications avoid:
- Implementation details (unless user-observable)
- Internal function names
- Code structure decisions

### Test Scenario Format

Test scenarios listed as:
```
1. **Scenario description**: Expected behavior
2. **Another scenario**: What should happen
```

Organized into:
- **Existing Tests**: What's currently tested
- **Additional Test Scenarios**: Recommended additions

### Command Documentation Structure

Each command spec includes:
1. **Command**: Syntax
2. **Purpose**: What it does (one line)
3. **Arguments**: Positional arguments
4. **Flags**: Optional flags
5. **Behavior**: Detailed behavior description
6. **Output Format**: stdout/stderr formatting
7. **Exit Codes**: Return values
8. **Edge Cases & Error Handling**: Special scenarios
9. **Examples**: Usage examples
10. **Testing Scenarios**: Test coverage
11. **Implementation Notes**: Key technical details
12. **Related Commands**: Cross-references

## Completeness

These specifications cover:
- ✅ All user-facing commands
- ✅ All command flags and options
- ✅ Plugin manifest format (complete)
- ✅ System-wide behaviors
- ✅ Error handling
- ✅ Edge cases
- ✅ Test scenarios
- ✅ Examples

## Implementation Notes

The specifications reference:
- **Go packages**: For context about implementation
- **Test files**: To show what's currently tested
- **Code patterns**: Where observable in behavior
- **Dependencies**: External libraries used

These references help but are not required for reimplementation.

## Version

These specifications document krew as of:
- **Git Commit**: 77a82ab (latest in repo)
- **Date**: January 2025
- **Go Version**: 1.25

## Contributing to Specs

When updating specifications:
1. Keep focus on observable behavior
2. Be precise and unambiguous
3. Include examples for clarity
4. Add test scenarios for new behavior
5. Update related specs for cross-cutting changes
6. Maintain consistent format across specs

## Questions or Issues

If specifications are unclear or incomplete:
1. Check related command specs for similar behavior
2. Review system-behavior.md for cross-cutting concerns
3. Examine integration tests for actual behavior
4. Consult code comments for edge cases

The goal is specifications clear enough to:
- Implement the system from scratch
- Verify existing implementation
- Create comprehensive tests
- Understand all user-facing behavior
