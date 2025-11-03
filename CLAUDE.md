# Claude Code Guide for TEVM Monorepo

This document provides guidance for working with Claude Code in the TEVM monorepo.

## CRITICAL: Working Directory Protocol

**Working directory is ALWAYS `/Users/williamcory/tevm` (monorepo root).**

### Absolute Path Rules

1. **NEVER use relative paths** - Always use absolute paths starting with `/Users/williamcory/tevm/`
2. **If you must `cd`**, ALWAYS return to root in the same command: `cd DIR && command && cd /Users/williamcory/tevm`
3. **Preferred: Use subshells** - `(cd DIR && command)` - automatically returns to original directory
4. **File operations** - Always use absolute paths: `/Users/williamcory/tevm/guillotine/src/file.zig`

### Examples

```bash
# ✅ CORRECT - subshell (preferred)
(cd /Users/williamcory/tevm/guillotine && cargo build --release)

# ✅ CORRECT - explicit return to root
cd /Users/williamcory/tevm/guillotine && zig build && cd /Users/williamcory/tevm

# ✅ CORRECT - absolute path
/Users/williamcory/tevm/guillotine/src/instructions/Stack.zig

# ❌ WRONG - relative cd (where are we now?)
cd guillotine

# ❌ WRONG - relative path (which submodule?)
src/instructions/Stack.zig
```

### Why This Matters

- **Prevents confusion** about current directory state
- **Makes paths unambiguous** - always know which submodule
- **Avoids errors** from being in wrong directory
- **Enables parallel work** across submodules

## Repository Structure

This is a monorepo managed with git submodules. Each subdirectory is an independent repository:

- `chains/` - Chain configurations
- `chop/` - Code utilities
- `compiler/` - Compiler tooling
- `guillotine/` - Execution framework ← **merge target**
- `guillotine-mini/` - Lightweight execution ← **merge source**
- `guillotine-rs/` - Rust implementation
- `json-rpc/` - RPC implementations
- `tevm-monorepo/` - Main monorepo
- `voltaire/` - Additional tooling
- `z-ens-normalize/` - ENS utilities

## Working with Submodules

When making changes:

1. **Navigate to the specific submodule** - Each submodule is its own git repository
2. **Make changes within the submodule** - Commits are made to the submodule's repository
3. **Update the parent repo** - The parent tracks which commit each submodule points to

### Common Commands

```bash
# Update all submodules to latest
git submodule update --remote

# Update a specific submodule
git submodule update --remote <submodule-name>

# Check status across all submodules
git submodule foreach git status

# Pull latest changes in all submodules
git submodule foreach git pull origin main
```

## Development Workflow

1. **Navigate to the project** you want to work on
2. **Check the project's README** for specific build/test instructions
3. **Make changes** within that submodule
4. **Test locally** using that project's test suite
5. **Commit to the submodule** repository
6. **Update parent** if needed to track the new commit

## Test-Driven Development (TDD) Protocol

**MANDATORY for all merge work: Write tests FIRST, then implement.**

### TDD Cycle

1. **Write failing test** - Define expected behavior in `.test.zig` file
2. **Verify it fails** - Run test, confirm it fails for the right reason
3. **Implement minimal code** - Make the test pass with simplest solution
4. **Verify it passes** - Run test, confirm it passes
5. **Refactor if needed** - Improve code while keeping tests green
6. **Commit progress** - Use commit command after solid progress

### Test File Organization

- **Test file naming**: `FileName.test.zig` (matches `FileName.zig`)
- **Test location**: Same directory as implementation
- **Example**:
  - Implementation: `/Users/williamcory/tevm/guillotine/src/instructions/Stack.zig`
  - Tests: `/Users/williamcory/tevm/guillotine/src/instructions/Stack.test.zig`

### Running Tests

```bash
# Run all tests
(cd /Users/williamcory/tevm/guillotine && zig build test)

# Run specific tests
(cd /Users/williamcory/tevm/guillotine && zig build test-unit -Dtest-filter='Stack')

# Verify build still works
(cd /Users/williamcory/tevm/guillotine && zig build)
```

### Committing Progress

After each completed phase or solid progress:
```bash
# Use commit command from chop
# See: /Users/williamcory/tevm/chop/.claude/commands/commit.md
```

## Tips for Claude Code

- Each submodule has its own dependencies and tooling
- Check `package.json` or `Cargo.toml` in each project for available scripts
- Build systems may differ between projects (npm, pnpm, cargo, etc.)
- Some projects may have interdependencies - check their documentation
- **NEVER create new directories** - use existing structure
- **Follow TDD** - tests first, then implementation

## Project Overview

### TypeScript Projects
Most projects use TypeScript/Node.js with modern tooling (likely Vite, Vitest, or similar)

### Rust Projects
- `guillotine-rs/` - Uses Cargo for building and testing

### Testing
Each project has its own test suite. Run tests within the specific project directory.
