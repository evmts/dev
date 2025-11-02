# Claude Code Guide for TEVM Monorepo

This document provides guidance for working with Claude Code in the TEVM monorepo.

## Repository Structure

This is a monorepo managed with git submodules. Each subdirectory is an independent repository:

- `chains/` - Chain configurations
- `chop/` - Code utilities
- `compiler/` - Compiler tooling
- `guillotine/` - Execution framework
- `guillotine-mini/` - Lightweight execution
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

## Tips for Claude Code

- Each submodule has its own dependencies and tooling
- Check `package.json` or `Cargo.toml` in each project for available scripts
- Build systems may differ between projects (npm, pnpm, cargo, etc.)
- Some projects may have interdependencies - check their documentation

## Project Overview

### TypeScript Projects
Most projects use TypeScript/Node.js with modern tooling (likely Vite, Vitest, or similar)

### Rust Projects
- `guillotine-rs/` - Uses Cargo for building and testing

### Testing
Each project has its own test suite. Run tests within the specific project directory.
