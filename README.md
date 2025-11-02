# TEVM Monorepo

This is a monorepo containing all TEVM (TypeScript Ethereum Virtual Machine) related projects.

## Projects

This monorepo contains the following submodules:

- **chains** - Chain configurations and definitions
- **chop** - Code chopping utilities
- **compiler** - Solidity/EVM compiler tooling
- **guillotine** - EVM execution and testing framework
- **guillotine-mini** - Lightweight version of guillotine
- **guillotine-rs** - Rust implementation of guillotine
- **json-rpc** - JSON-RPC client and server implementations
- **tevm-monorepo** - Main TEVM monorepo
- **voltaire** - Additional TEVM tooling
- **z-ens-normalize** - ENS name normalization utilities

## Getting Started

Clone this repository with submodules:

```bash
git clone --recurse-submodules https://github.com/evmts/tevm.git
```

Or if you've already cloned it, initialize the submodules:

```bash
git submodule update --init --recursive
```

## Development

Each submodule is an independent project with its own dependencies and build system. Navigate to individual project directories to work on them.

## Structure

This repository uses git submodules to manage the individual projects. Each directory represents a separate git repository that can be developed independently while being part of the larger monorepo structure.
