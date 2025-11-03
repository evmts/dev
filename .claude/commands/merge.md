# Merge guillotine-mini into guillotine

## Description

Merge the @guillotine-mini/ submodule into @guillotine/ submodule.

## Prompt

````xml
<task>
  <objective long-term>
    Merge the guillotine-mini submodule into the guillotine submodule
  </objective>
  <context>
    <source-repo guillotine-mini>
      <description>
        ⚠️ GUILLOTINE-MINI IS ALWAYS THE SOURCE OF TRUTH ⚠️

        Full documentation: @guillotine-mini/CLAUDE.md

        Key files to reference:
        - @guillotine-mini/src/instructions/handlers_arithmetic.zig
        - @guillotine-mini/src/instructions/handlers_bitwise.zig
        - @guillotine-mini/src/instructions/handlers_comparison.zig
        - @guillotine-mini/src/instructions/handlers_stack.zig
        - @guillotine-mini/src/frame.zig (for Frame interface patterns)
        - @guillotine-mini/src/evm.zig (for state management patterns)

        Why guillotine-mini is the source of truth:
        - Passes all ethereum/tests GeneralStateTests
        - Spec-compliant implementation (verified against execution-specs)
        - Simple, correct, well-tested

        When copying: TRUST the implementation. If your tests fail, YOU copied wrong.
      </description>
    </source-repo>

    <target-repo guillotine>
      <description>
        Target repository for merged implementation.

        Full documentation: @guillotine/CLAUDE.md
        Architecture: @guillotine/src/instructions/CLAUDE.md

        Key differences from guillotine-mini:
        - Dispatch-based execution (not PC-based)
        - Optimized for performance
        - Supports synthetic fused opcodes
        - More complex but faster

        Merge goal: Create shared instruction implementations that work with BOTH architectures.
      </description>
    </target-repo>

  </context>

  <source-priority>
    guillotine-mini is the production-ready, spec-compliant EVM. When logic or tests differ between implementations, use guillotine-mini as source of truth. It passes official Ethereum execution specs.

```bash
zig build              # Build all modules
zig build test         # Run unit + spec tests
zig build specs        # Run ethereum/tests validation
zig build wasm         # Build WebAssembly library
zig build test-watch   # Interactive test runner

# Debug failing tests (RECOMMENDED)
bun scripts/isolate-test.ts "test_name"  # Max debug output + analysis
bun scripts/test-subset.ts "pattern"     # Filter test categories
````

**Prerequisites**:

- **Zig 0.15.1+** (core build system)
- **Cargo** (Rust package manager - **required** for building BN254/ARK cryptographic dependencies)
- **Python 3.8+** (for test generation and reference implementation)
- **uv** (Python package manager for spec fixture generation): `brew install uv`
- **Bun** (for TS helpers/agents): `brew install bun`

---

## Architecture

```
src/
├── evm.zig               # Orchestrator: state, storage, gas refunds, nested calls
├── frame.zig             # Bytecode interpreter: stack, memory, PC, per-opcode logic
├── host.zig              # Abstract state backend interface
├── hardfork.zig          # Hardfork detection and feature flags
├── opcode.zig            # Opcode definitions and utilities
├── trace.zig             # EIP-3155 trace generation
└── errors.zig            # Error types

External Dependencies (fetched via zig build):
├── primitives            # Ethereum types (Address, u256, gas constants, RLP, ABI, etc.)
│                         # Source: https://github.com/evmts/primitives
├── crypto                # Cryptographic primitives (keccak256, secp256k1, BLS12-381)
└── precompiles           # Ethereum precompiled contracts
```

### Key Design Principles

| Component    | Responsibility                                                           |
| ------------ | ------------------------------------------------------------------------ |
| **Evm**      | State management, storage, gas refunds, warm/cold tracking, nested calls |
| **Frame**    | Stack, memory, PC, bytecode interpretation, opcode execution             |
| **Host**     | Pluggable state backend (balances, nonces, code, logs, self-destruct)    |
| **Hardfork** | Gas cost adjustments, feature flag guards (`isAtLeast()`, `isBefore()`)  |

**Allocation strategy**: Arena allocator for transaction-scoped memory (all freed at transaction end)

---

## Testing & Debugging

### Test Types

| Type               | Purpose                           | Command                                  |
| ------------------ | --------------------------------- | ---------------------------------------- |
| **Unit tests**     | Inline `test` blocks              | `zig build test`                         |
| **Spec tests**     | ethereum/tests GeneralStateTests  | `zig build specs`                        |
| **Filtered tests** | By hardfork/EIP/opcode            | `TEST_FILTER="Cancun" zig build specs`   |
| **Trace tests**    | EIP-3155 trace capture/comparison | `zig build test-trace`                   |
| **Watch mode**     | Auto-reload on changes            | `zig build test-watch`                   |
| **Engine tests**   | Consensus-layer format tests      | `INCLUDE_ENGINE_TESTS=1 zig build specs` |

**Prerequisites:**

- Zig 0.15.1 or later
- Python 3.8+ (for test generation and reference implementation)
- uv (Python package manager) for spec fixture generation: `brew install uv`
- Bun (for TS helpers/agents): `brew install bun`

### Test Scope and Filtering

**Engine API Tests (Optional)**

By default, `blockchain_test_engine` format tests are **disabled** because they test consensus-layer functionality (block validation, Engine API payloads) rather than core EVM execution. These tests are out of scope for an EVM library like guillotine-mini (similar to how REVM doesn't implement Engine API).

To include them for comprehensive testing:

```bash
INCLUDE_ENGINE_TESTS=1 zig build specs
```

**What's tested:**

- ✅ **Default**: Pure EVM execution (opcodes, gas, state transitions, precompiles)
- ✅ **Default**: Transaction processing and hardfork-specific EVM changes
- ❌ **Disabled**: Block validation, consensus rules, Engine API server implementation
- ❌ **Disabled**: Withdrawal timing edge cases (consensus-layer concern)

### Helper Scripts

<details>
<summary><b>🔬 isolate-test.ts</b> - Test Isolation Helper (⭐ RECOMMENDED)</summary>

```bash
bun scripts/isolate-test.ts "exact_test_name"
```

**Features:**

- Runs single test with maximum debug output
- Automatic failure type detection (crash/gas/behavior)
- Trace divergence analysis (PC, opcode, gas, stack)
- Next-step debugging guidance
- Quick reference commands

</details>

<details>
<summary><b>🎯 test-subset.ts</b> - Test Subset Runner</summary>

# Using helper scripts

bun scripts/test-subset.ts Cancun
bun scripts/test-subset.ts transientStorage
bun scripts/test-subset.ts MCOPY

# Or using shell scripts

./scripts/test-subset.sh Cancun
./scripts/test-subset.sh "exact_test_name"

# Direct filtering with zig build

TEST_FILTER="Cancun" zig build specs
TEST_FILTER="transientStorage" zig build specs
TEST_FILTER="push0" zig build specs

````

#### Granular Spec Targets

Large hardforks are split into smaller sub-targets for faster iteration:

```bash
# Berlin
zig build specs-berlin-acl
zig build specs-berlin-intrinsic-gas-cost
zig build specs-berlin-intrinsic-type0
zig build specs-berlin-intrinsic-type1

# Frontier
zig build specs-frontier-precompiles
zig build specs-frontier-identity
zig build specs-frontier-create
zig build specs-frontier-call
zig build specs-frontier-calldata
zig build specs-frontier-dup
zig build specs-frontier-push
zig build specs-frontier-stack
zig build specs-frontier-opcodes

# Shanghai
zig build specs-shanghai-push0
zig build specs-shanghai-warmcoinbase
zig build specs-shanghai-initcode
zig build specs-shanghai-withdrawals

# Cancun
zig build specs-cancun-tstore-basic
zig build specs-cancun-tstore-reentrancy
zig build specs-cancun-tstore-contexts
zig build specs-cancun-mcopy
zig build specs-cancun-selfdestruct
zig build specs-cancun-blobbasefee
zig build specs-cancun-blob-precompile
zig build specs-cancun-blob-opcodes
zig build specs-cancun-blob-tx-small
zig build specs-cancun-blob-tx-subtraction
zig build specs-cancun-blob-tx-insufficient
zig build specs-cancun-blob-tx-sufficient
zig build specs-cancun-blob-tx-valid-combos

# Prague
zig build specs-prague-calldata-cost-type0
zig build specs-prague-calldata-cost-type1-2
zig build specs-prague-calldata-cost-type3
zig build specs-prague-calldata-cost-type4
zig build specs-prague-calldata-cost-refunds
zig build specs-prague-bls-g1
zig build specs-prague-bls-g2
zig build specs-prague-bls-pairing
zig build specs-prague-bls-map
zig build specs-prague-bls-misc
zig build specs-prague-setcode-calls
zig build specs-prague-setcode-gas
zig build specs-prague-setcode-txs
zig build specs-prague-setcode-advanced

# Osaka
zig build specs-osaka-modexp-variable-gas
zig build specs-osaka-modexp-vectors-eip
zig build specs-osaka-modexp-vectors-legacy
zig build specs-osaka-modexp-misc
zig build specs-osaka-other
````

**Use for:** Running entire test categories by hardfork, EIP, or pattern

</details>

<details>
<summary><b>🤖 fix-specs.ts</b> - Automated Spec Fixer</summary>

```bash
bun run scripts/fix-specs.ts               # Fix all test suites
bun run scripts/fix-specs.ts suite <name>  # Fix specific suite
```

**Features:**

- AI-powered systematic test fixing
- 7-checkpoint methodology (no guesswork)
- Historical context injection (known-issues.json)
- Auto-commit on success
- Comprehensive reporting (reports/spec-fixes/)

</details>

### Debugging Workflow

```bash
# 1. Find failures
bun scripts/test-subset.ts transientStorage

# 2. Isolate and debug (get automated analysis)
bun scripts/isolate-test.ts "transStorageReset"

# 3. Review divergence output (PC, opcode, gas, stack)
# 4. Fix in src/frame.zig or src/evm.zig
# 5. Verify
bun scripts/isolate-test.ts "transStorageReset"
```

**Test filtering patterns:**

- **Hardfork**: `Cancun`, `Shanghai`, `London`, `Berlin`, `Merge`, `Prague`
- **EIP**: `transientStorage`, `push0`, `MCOPY`, `warmcoinbase`
- **Opcode**: `add`, `mul`, `sstore`, `sload`, `call`, `create2`
- **Category**: `vmArithmeticTest`, `vmBitwiseLogicOperation`, `vmIOandFlowOperations`

---

### Spec Fixer (AI-Assisted)

Use an agent to iterate on failing specs and generate focused reports:

Prereqs:

- `bun` installed and `cd scripts && bun install`
- `ANTHROPIC_API_KEY` exported in your shell (or `.env` at repo root)

Run:

```bash
# All suites
bun run scripts/fix-specs.ts

# One suite
bun run scripts/fix-specs.ts suite shanghai-push0
```

The script runs tests, invokes the agent on failure, and saves reports in `reports/spec-fixes/` with a summary at the end. If no API key is set, it skips auto-fix and just runs the tests.

## Core Components

<details>
<summary><b>Evm (src/evm.zig)</b></summary>

**Orchestrates:**

- State management (storage, balances, nonces, code)
- Call stack (nested CALL/CREATE, max depth 1024)
- Gas accounting (refunds, warm/cold access via EIP-2929)
- Transient storage (EIP-1153, transaction-scoped)
- Hardfork rules (automatic gas adjustment)

**Key methods:**

- `call()` - Main entry point
- `inner_call()` - CALL/STATICCALL/DELEGATECALL
- `inner_create()` - CREATE/CREATE2
- `accessAddress()`, `accessStorageSlot()` - EIP-2929 tracking
- `get_storage()`, `set_storage()` - Persistent storage
- `get_transient_storage()`, `set_transient_storage()` - EIP-1153

</details>

<details>
<summary><b>Frame (src/frame.zig)</b></summary>

**Manages single execution context:**

- 256-word stack (ArrayList of u256)
- Expandable memory (byte array)
- Program counter (PC) and gas tracking
- Bytecode interpretation

**Key methods:**

- `execute()` - Main execution loop
- `step()` - Single instruction (for tracing)
- Opcode implementations (arithmetic, bitwise, storage, control flow, calls)

</details>

<details>
<summary><b>Host Interface (src/host.zig)</b></summary>

Abstract interface for external state access:

```zig
pub const HostInterface = struct {
    ptr: *anyopaque,
    vtable: *const VTable,
    pub const VTable = struct {
        getBalance: *const fn (ptr: *anyopaque, address: Address) u256,
        setBalance: *const fn (ptr: *anyopaque, address: Address, balance: u256) void,
        getCode: *const fn (ptr: *anyopaque, address: Address) []const u8,
        setCode: *const fn (ptr: *anyopaque, address: Address, code: []const u8) void,
        getStorage: *const fn (ptr: *anyopaque, address: Address, slot: u256) u256,
        setStorage: *const fn (ptr: *anyopaque, address: Address, slot: u256, value: u256) void,
        getNonce: *const fn (ptr: *anyopaque, address: Address) u64,
        setNonce: *const fn (ptr: *anyopaque, address: Address, nonce: u64) void,
    };
};
```

**Note:** The host interface is used only for external state backends. Nested calls (CALL, DELEGATECALL, etc.) are handled internally by `Evm.inner_call()` and do not use this interface.

Test implementation: `test/specs/test_host.zig`

</details>

<details>
<summary><b>Hardfork Support (src/hardfork.zig)</b></summary>

```zig
pub const Hardfork = enum(u8) {
    FRONTIER, HOMESTEAD, TANGERINE, SPURIOUS, BYZANTIUM,
    CONSTANTINOPLE, ISTANBUL, BERLIN, LONDON, MERGE,
    SHANGHAI, CANCUN, PRAGUE,
    pub const DEFAULT = Hardfork.CANCUN;
};
```

**Methods:** `isAtLeast(fork)`, `isBefore(fork)`, `fromString(name)`

</details>

<details>
<summary><b>Primitives Module (External Dependency)</b></summary>

The primitives library is now an external dependency fetched via `zig fetch` from https://github.com/evmts/primitives. It is no longer included as a git submodule.

**Modules provided:**

- **Address** - 20-byte Ethereum address
- **Uint(N)** - Arbitrary precision (u256, u512)
- **GasConstants** - Per-opcode costs, hardfork-aware
- **Rlp** - RLP encoding/decoding
- **Abi** - ABI encoding/decoding
- **Transaction** - Legacy, EIP-2930, EIP-1559, EIP-4844
- **AccessList** - EIP-2930 support
- **Blob** - EIP-4844 blob transactions
- **Hex** - Hex encoding/decoding
- **Crypto** - Cryptographic primitives (keccak256, secp256k1, BLS12-381)
- **Precompiles** - Ethereum precompiled contracts

</details>

---

## EIP Support

| EIP      | Feature                                  | Hardfork | Status |
| -------- | ---------------------------------------- | -------- | ------ |
| EIP-2929 | State access gas costs                   | Berlin   | ✅     |
| EIP-2930 | Access lists                             | Berlin   | ✅     |
| EIP-1559 | Fee market                               | London   | ✅     |
| EIP-3198 | BASEFEE opcode                           | London   | ✅     |
| EIP-3529 | Reduced gas refunds                      | London   | ✅     |
| EIP-3541 | Reject code starting with 0xEF           | London   | ✅     |
| EIP-3651 | Warm coinbase                            | Shanghai | ✅     |
| EIP-3855 | PUSH0 instruction                        | Shanghai | ✅     |
| EIP-3860 | Limit init code size                     | Shanghai | ✅     |
| EIP-1153 | Transient storage (TLOAD/TSTORE)         | Cancun   | ✅     |
| EIP-4844 | Blob transactions (BLOBHASH/BLOBBASEFEE) | Cancun   | ✅     |
| EIP-5656 | MCOPY instruction                        | Cancun   | ✅     |
| EIP-6780 | SELFDESTRUCT only in same tx             | Cancun   | ✅     |
| EIP-7516 | BLOBBASEFEE opcode                       | Cancun   | ✅     |

---

## Gas Metering

**Location:** `src/primitives/gas_constants.zig`

- **Base costs** - Per-opcode execution (ADD, MUL, SSTORE)
- **Memory expansion** - Quadratic cost for growth
- **Call stipend** - 2300 gas for value transfers
- **Warm/Cold access** - EIP-2929 (Berlin+): warm=100, cold=2600
- **Gas refunds** - Capped at 1/2 (pre-London) or 1/5 (London+)
- **Intrinsic gas** - 21000 + calldata costs

---

## Tracing

Full EIP-3155 trace support:

```bash
zig build test-trace
```

**Includes:** PC, opcode, gas remaining, stack, memory, storage changes

Compare traces against reference implementations (geth, execution-specs) to identify divergences.

---

## WASM Build

```bash
zig build wasm
# Output: zig-out/bin/guillotine_mini.wasm (~100-200 KB)
```

<details>
<summary><b>Exported C API Functions</b></summary>

- `evm_create()`, `evm_destroy()`
- `evm_set_bytecode()`, `evm_set_execution_context()`, `evm_set_blockchain_context()`
- `evm_execute()`, `evm_get_gas_remaining()`, `evm_get_gas_used()`
- `evm_is_success()`, `evm_get_output_len()`, `evm_get_output()`
- `evm_set_storage()`, `evm_get_storage()`, `evm_set_balance()`, `evm_set_code()`

See `src/root_c.zig` for full API.

</details>

---

## Development Workflow

### Build Targets

| Command                | Purpose                    |
| ---------------------- | -------------------------- |
| `zig build`            | Build all modules          |
| `zig build test`       | Unit + spec tests          |
| `zig build specs`      | Spec tests only            |
| `zig build wasm`       | WASM library + size report |
| `zig build test-watch` | Interactive test runner    |
| `zig build test-trace` | Trace capture tests        |

### Adding New Features

1. **Implement**: `src/frame.zig` (opcodes) or `src/evm.zig` (behavior)
2. **Update gas**: `src/primitives/gas_constants.zig` if needed
3. **Add guards**: `self.hardfork.isAtLeast(.FORK_NAME)`
4. **Test**: Run relevant spec tests
5. **Debug**: Use trace divergence analysis

### Code Style

- **Naming**: `snake_case` (functions/variables), `PascalCase` (types)
- **Errors**: Explicit unions, propagate with `try`
- **Comments**: Explain "why", not "what"
- **Docs**: Use `///` for public APIs
- **Format**: Run `zig fmt` before commit

---

## Agent Quick Reference: Python Reference vs Zig Implementation

> **Critical for debugging**: Python execution-specs are the authoritative source. When in doubt, trust Python code over intuition, docs, or Yellow Paper.

> **⚠️ IMPORTANT**: The `execution-specs/` directory is a git submodule containing the official Ethereum execution specifications. **NEVER commit, clean, or modify any files within this submodule.** It should be managed separately and left untouched during normal development.

### File Location Mapping

| Problem Area        | Python Reference                                 | Zig Implementation                                   |
| ------------------- | ------------------------------------------------ | ---------------------------------------------------- |
| Opcode logic        | `execution-specs/.../vm/instructions/*.py`       | `src/frame.zig`                                      |
| Gas calculation     | `execution-specs/.../vm/gas.py`                  | `src/primitives/gas_constants.zig` + `src/frame.zig` |
| Call/Create         | `execution-specs/.../vm/instructions/system.py`  | `src/evm.zig` (`inner_call`, `inner_create`)         |
| Storage ops         | `execution-specs/.../vm/instructions/storage.py` | `src/evm.zig` (get/set storage)                      |
| Transient storage   | `execution-specs/forks/cancun/.../storage.py`    | `src/evm.zig` (get/set transient)                    |
| State management    | `execution-specs/.../state.py`                   | `src/evm.zig` (balances, nonces, code)               |
| Hardfork activation | `execution-specs/forks/<hardfork>/`              | `src/hardfork.zig`                                   |
| Warm/cold tracking  | `execution-specs/.../vm/__init__.py`             | `src/evm.zig` (warm_addresses, warm_storage_slots)   |
| Memory ops          | `execution-specs/.../vm/memory.py`               | `src/frame.zig` (expandMemory)                       |
| Stack ops           | `execution-specs/.../vm/stack.py`                | `src/frame.zig` (pushStack, popStack)                |

### Architectural Differences

**Python: Single `Evm` Class**

- Location: `execution-specs/.../vm/__init__.py`
- One dataclass: `evm.stack`, `evm.memory`, `evm.pc`, `evm.gas_left`, `evm.message.block_env.state`

**Zig: Split `Evm` + `Frame`**

- **Evm** (`src/evm.zig`): State, storage, gas refunds, nested calls
  - Storage maps: `storage`, `transient_storage`, `original_storage`
  - Warm/cold tracking: `warm_addresses`, `warm_storage_slots`
  - Gas: `gas_refund`
- **Frame** (`src/frame.zig`): Single call frame execution
  - Stack, memory, PC, gas, bytecode
  - Per-frame context: caller, address, value, calldata

> **Key insight**: Python's `evm.stack` → Zig's `frame.stack`. Python's `evm.message.block_env.state` → Zig's `evm.storage`/`evm.balances`

---

## Common Bug Patterns

<details>
<summary><b>1. Gas Metering Bugs (SSTORE example)</b></summary>

**Python pattern:**

```python
def sstore(evm: Evm) -> None:
    key = pop(evm.stack).to_be_bytes32()
    new_value = pop(evm.stack)

    # Check gas stipend FIRST
    if evm.gas_left <= GAS_CALL_STIPEND:
        raise OutOfGasError

    # Then calculate dynamic cost
    gas_cost = Uint(0)
    if (target, key) not in evm.accessed_storage_keys:
        evm.accessed_storage_keys.add((target, key))
        gas_cost += GAS_COLD_SLOAD

    if original_value == current_value and current_value != new_value:
        if original_value == 0:
            gas_cost += GAS_STORAGE_SET
        else:
            gas_cost += GAS_STORAGE_UPDATE - GAS_COLD_SLOAD
    else:
        gas_cost += GAS_WARM_ACCESS

    charge_gas(evm, gas_cost)
```

**Common mistakes:**

- ❌ Forgetting `SstoreSentryGas` (2300) check
- ❌ Not tracking `original_storage` separately from `storage`
- ❌ Wrong gas refund calculation
- ❌ Not adding cold access cost before warm/set/update

</details>

<details>
<summary><b>2. Warm/Cold Access Tracking (EIP-2929)</b></summary>

**Python:**

```python
accessed_addresses: Set[Address]
accessed_storage_keys: Set[Tuple[Address, Bytes]]

if address not in evm.accessed_addresses:
    evm.accessed_addresses.add(address)
    charge_gas(evm, GAS_COLD_ACCOUNT_ACCESS)
else:
    charge_gas(evm, GAS_WARM_ACCESS)
```

**Zig:**

```zig
warm_addresses: ArrayHashMap(Address, void, AddressContext, false)
warm_storage_slots: ArrayHashMap(StorageSlotKey, void, StorageSlotKeyContext, false)

if (!self.warm_addresses.contains(address)) {
    try self.warm_addresses.put(address, {});
    return ColdAccountAccessCost;
} else {
    return WarmStorageReadCost;
}
```

**Key difference**: Python uses sets, Zig uses hash maps with `void` values.

</details>

<details>
<summary><b>3. Transient Storage (EIP-1153)</b></summary>

**Python:**

```python
def tload(evm: Evm) -> None:
    key = pop(evm.stack).to_be_bytes32()
    charge_gas(evm, GAS_WARM_ACCESS)  # Always warm, never cold
    value = get_transient_storage(...)
    push(evm.stack, value)

def tstore(evm: Evm) -> None:
    key = pop(evm.stack).to_be_bytes32()
    new_value = pop(evm.stack)
    charge_gas(evm, GAS_WARM_ACCESS)  # Always warm
    if evm.message.is_static:
        raise WriteInStaticContext
    set_transient_storage(...)
```

**Critical rules:**

- ✅ Transient storage is ALWAYS warm (100 gas), never cold
- ✅ Cleared at transaction boundaries, NOT call boundaries
- ✅ Must check `is_static` for TSTORE
- ✅ No gas refunds

</details>

<details>
<summary><b>4. Hardfork-Specific Behavior</b></summary>

**Python**: Separate directories (`execution-specs/forks/berlin/`, `cancun/`), each fork inherits and overrides.

**Zig**: Runtime checks in one codebase:

```zig
if (self.hardfork.isAtLeast(.CANCUN)) {
    // Cancun-specific (EIP-1153, EIP-4844)
} else if (self.hardfork.isAtLeast(.SHANGHAI)) {
    // Shanghai-specific (PUSH0, warm coinbase)
}
```

**Common mistakes:**

- ❌ Wrong hardfork for feature (e.g., PUSH0 before Shanghai)
- ❌ Not using `isAtLeast` for backward compatibility
- ❌ Breaking earlier hardforks when adding new feature

</details>

---

## Gas Constant Reference

> **Must match exactly** between Python and Zig.

<details>
<summary><b>Gas Constants Table</b></summary>

| Operation                       | Python                     | Zig                     | Value | Hardfork |
| ------------------------------- | -------------------------- | ----------------------- | ----- | -------- |
| Warm storage read               | `GAS_WARM_ACCESS`          | `WarmStorageReadCost`   | 100   | Berlin+  |
| Cold SLOAD                      | `GAS_COLD_SLOAD`           | `ColdSloadCost`         | 2100  | Berlin+  |
| Cold account access             | `GAS_COLD_ACCOUNT_ACCESS`  | `ColdAccountAccessCost` | 2600  | Berlin+  |
| SSTORE set (0→nonzero)          | `GAS_STORAGE_SET`          | `SstoreSetGas`          | 20000 | All      |
| SSTORE update (nonzero→nonzero) | `GAS_STORAGE_UPDATE`       | `SstoreResetGas`        | 5000  | All      |
| SSTORE clear refund             | `GAS_STORAGE_CLEAR_REFUND` | `SstoreClearRefund`     | 4800  | London+  |
| SSTORE stipend check            | `GAS_CALL_STIPEND`         | `SstoreSentryGas`       | 2300  | All      |
| Call value transfer             | `GAS_CALL_VALUE`           | `CallValueCost`         | 9000  | All      |
| Call stipend                    | `GAS_CALL_STIPEND`         | `CallStipend`           | 2300  | All      |

**Locations:**

- Python: `execution-specs/src/ethereum/forks/<hardfork>/vm/gas.py`
- Zig: `src/primitives/gas_constants.zig`

</details>

---

## Debugging Workflow Cheat Sheet

**When a test fails:**

1. **Run with trace**

   ```bash
   TEST_FILTER="exact_test_name" zig build specs
   # Or use: bun scripts/isolate-test.ts "exact_test_name"
   ```

2. **Identify divergence**

   - Find "Trace divergence at step N"
   - Note: PC, opcode, gas remaining, stack state

3. **Find Python reference**

   ```bash
   cd execution-specs/src/ethereum/forks/cancun/vm/instructions/
   grep -r "def <opcode_name>" .
   ```

4. **Read Python implementation**

   - Note gas charge order (matters!)
   - Note state modifications
   - Note refund updates
   - Note error conditions

5. **Compare Zig implementation**

   - Opcodes: `src/frame.zig`
   - Calls/creates: `src/evm.zig`
   - Storage: `src/evm.zig`

6. **Fix minimally**

   - Match Python exactly
   - Keep gas calculation order identical
   - Preserve hardfork guards

7. **Verify**
   ```bash
   TEST_FILTER="exact_test_name" zig build specs
   ```

---

## Pro Tips

1. **Gas calculation order matters** - Match Python's exact sequence (e.g., SSTORE: stipend → cold access → value comparison → refunds)
2. **Original vs Current storage** - Track both for refund calculations
3. **Warm/cold is cumulative** - Once warm, stays warm for entire transaction
4. **Refund counter can go negative** - Gets capped at transaction end, don't clamp prematurely
5. **Static context propagates** - STATICCALL → all child calls inherit `is_static`
6. **Memory expansion is quadratic** - `size_in_words ** 2 // 512` (match Python exactly)
7. **Call depth limit is 1024** - Check before any CALL/CREATE

---

## Anti-Patterns

> **Things NOT to do:**

- ❌ Guess gas costs (must match Python exactly)
- ❌ Skip trace comparison (shows exact divergence point)
- ❌ Mix hardfork behaviors (use `isAtLeast` guards)
- ❌ Ignore error conditions (Python's `raise` must map to Zig errors)
- ❌ Hardcode test-specific logic (fix general implementation)
- ❌ Forget to charge gas before operations (Python charges first)
- ❌ Modify test files (only change `src/` implementations)
- ❌ **CRITICAL: Silently ignore errors with `catch {}`** - ALL errors MUST be handled and/or propagated properly. Never use `catch {}` to suppress errors (e.g., `slots_to_remove.append(allocator, key) catch {}` is forbidden). Either handle the error meaningfully or propagate it with `try`. The only exception is when the function signature explicitly cannot return errors (non-error-union return type), in which case the function design should be reconsidered.
- ❌ **NEVER create .backup files** - We use git for version control. Never create files with .backup, .bak, .old, or similar extensions. Use git branches/stashes instead.

---

## Essential Python Reference Files

**Cancun (latest tested hardfork):**

- `execution-specs/src/ethereum/forks/cancun/vm/interpreter.py` - Main loop
- `execution-specs/src/ethereum/forks/cancun/vm/gas.py` - Gas calculations
- `execution-specs/src/ethereum/forks/cancun/vm/instructions/storage.py` - SLOAD/SSTORE/TLOAD/TSTORE
- `execution-specs/src/ethereum/forks/cancun/vm/instructions/system.py` - CALL/CREATE/SELFDESTRUCT
- `execution-specs/src/ethereum/forks/cancun/vm/instructions/arithmetic.py` - ADD/MUL/EXP
- `execution-specs/src/ethereum/forks/cancun/vm/instructions/memory.py` - MLOAD/MSTORE/MCOPY
- `execution-specs/src/ethereum/forks/cancun/state.py` - State primitives

**For other hardforks:** Replace `cancun` with `shanghai`, `paris`, `london`, `berlin`, etc.

---

## 🤖 Automated Spec Fixer: 7-Checkpoint System

The `scripts/fix-specs.ts` pipeline enforces a **mandatory 7-checkpoint methodology** for systematic, evidence-based debugging.

### Checkpoints

Each checkpoint requires **actual data** (no placeholders):

1. **✅ Run Test and Confirm Failure** - Execute, capture output, identify failing tests
2. **✅ Generate Trace Comparison** - Use `bun scripts/isolate-test.ts`, identify divergence (PC, opcode, gas, stack)
3. **✅ Read Python Reference** - Navigate to `execution-specs/.../`, quote actual code (not summaries)
4. **✅ Compare Zig Implementation** - Read current code, quote actual Zig, identify discrepancies
5. **✅ Diagnose Root Cause and Propose Fix** - Based on checkpoints 2-4, propose specific fix
6. **✅ Implement Fix** - Make minimal changes, preserve hardfork compatibility
7. **✅ Verify Fix** - Re-run test, confirm pass (if failing, return to Checkpoint 2)

### Enforcement Rules

- ✅ ALL checkpoints MUST be explicitly confirmed with actual data
- ✅ NO placeholders: `[TODO]`, `[TBD]`, `[value]`
- ✅ Iteration required if tests fail
- ❌ DO NOT skip checkpoints
- ❌ DO NOT proceed to fixes without analysis (1-5)
- ❌ DO NOT use summaries instead of code quotes

### Usage

```bash
bun run scripts/fix-specs.ts               # Fix all suites
bun run scripts/fix-specs.ts suite <name>  # Fix specific suite

# Reports: reports/spec-fixes/
#   - pipeline-summary.md: Overall results
#   - pipeline-summary-ai.md: AI narrative summary
#   - {suite}-attempt{N}.md: Per-suite debugging reports
```

### Known Issues Database

**Location:** `scripts/known-issues.json`

Tracks historical debugging insights:

- Common failure patterns and root causes
- Relevant file locations (with line ranges)
- Python reference locations
- Key invariants
- Expected gas costs

**Schema:**

```json
{
  "test-suite-name": {
    "test_suite": "test-suite-name",
    "description": "Brief description",
    "common_causes": ["Cause 1", "Cause 2"],
    "relevant_files": ["src/file.zig:function", "src/other.zig:line-range"],
    "python_ref": "execution-specs/.../reference.py",
    "key_invariants": ["Invariant 1", "Invariant 2"],
    "gas_costs": { "OPERATION": 100 }
  }
}
```

---

## Resources

- **ethereum/tests**: https://github.com/ethereum/tests
- **execution-specs**: https://github.com/ethereum/execution-specs
- **EIP Index**: https://eips.ethereum.org/
- **EIP-3155 (Trace Format)**: https://eips.ethereum.org/EIPS/eip-3155
- **Yellow Paper**: https://ethereum.github.io/yellowpaper/paper.pdf
- **Zig Documentation**: https://ziglang.org/documentation/

---

## File Summary

<details>
<summary><b>Core Implementation Files</b></summary>

| File                      | Purpose              | Key Exports                                      |
| ------------------------- | -------------------- | ------------------------------------------------ |
| `src/root.zig`            | Main module          | Evm, Frame, Host, Hardfork, Tracer               |
| `src/evm.zig`             | EVM orchestrator     | Evm struct, call(), inner_call(), inner_create() |
| `src/frame.zig`           | Bytecode interpreter | Frame struct, execute(), step()                  |
| `src/host.zig`            | Host interface       | HostInterface, CallResult, CallType              |
| `src/hardfork.zig`        | Hardfork logic       | Hardfork enum, isAtLeast(), fromString()         |
| `src/opcode.zig`          | Opcode utilities     | getOpName(), opcode constants                    |
| `src/trace.zig`           | EIP-3155 tracing     | Tracer, TraceEntry                               |
| `src/errors.zig`          | Error types          | CallError enum                                   |
| `src/primitives/root.zig` | Primitives exports   | All primitive types                              |
| `build.zig`               | Build config         | Build targets, dependencies                      |
| `test_runner.zig`         | Test runner          | Test execution, reporting                        |
| `test/specs/runner.zig`   | Spec test execution  | runJsonTest(), trace comparison                  |

</details>

---

## Contributing

When making changes:

1. Ensure all tests pass (`zig build test`)
2. Run spec tests for affected hardfork
3. Format code (`zig fmt src/ test/`)
4. Add tests for new features
5. Update documentation

**For questions:** Refer to test output and trace divergence analysis.

**License:** See LICENSE file.
</source-repo>

    <target-repo guillotine>
      <description>The following is the guillotine claude.md</description>

# CLAUDE.md

## MISSION CRITICAL SOFTWARE

**⚠️ WARNING: Mission-critical financial infrastructure - bugs cause fund loss.**

Every line of code must be correct. Zero error tolerance.

## Core Protocols

### Working Directory

**ALWAYS run commands from the repository root directory.** Never use `cd` except when debugging a submodule. All commands, builds, and tests are designed to run from root.

### Security

- Sensitive data detected (API keys/passwords/tokens): abort, explain, request sanitized prompt
- Memory safety: plan ownership/deallocation for every allocation
- Every change must be tested and verified
- Use SafetyCounter for infinite loop prevention (300M instruction limit)
- **CRITICAL: Crashes are SEVERE SECURITY BUGS** - Any crash (e.g., from `std.debug.assert`) indicates memory unsafety or missing validation. The EVM must ALWAYS return errors gracefully, never crash. Before fixing the bug that triggered the crash, FIRST fix the validation/error handling that allowed the crash to occur.

### Build Verification

**EVERY code change**: `zig build && zig build test-opcodes`
**Exception**: .md files only

Follow TDD

### Debugging

- Bug not obvious = improve visibility first
- Use differential tests with revm in test/differential

### Zero Tolerance

❌ Broken builds/tests
❌ Stub implementations (`error.NotImplemented`)
❌ Commented code (use Git)
❌ Test failures
❌ Invalid benchmarks
❌ `std.debug.print` in modules (use `log.zig`)
❌ `std.debug.assert` (use `tracer.assert()`)
❌ Skipping/commenting tests
❌ Any stub/fallback implementations
❌ **Swallowing errors with `catch` (e.g., `catch {}`, `catch &.{}`, `catch null`)**

**STOP and ask for help rather than stubbing.**

**WHY PLACEHOLDERS ARE BANNED**: Placeholder implementations create ambiguity - the human cannot tell if "Coming soon!" or simplified output means:

1. The AI couldn't solve it and gave up
2. The AI is planning to implement it later
3. The feature genuinely isn't ready yet
4. There's a technical blocker

This uncertainty wastes debugging time and erodes trust. Either implement it fully, explain why it can't be done, or ask for help. Never leave placeholders that pretend to work.

**NEVER swallow errors! Every error must be explicitly handled or propagated. Using `catch` to ignore errors can cause silent failures and fund loss.**

## Coding Standards

### Principles

- Minimal else statements
- Single word variables (`n` not `number`)
- Direct imports (`address.Address` not aliases)
- Tests in source files
- Defer patterns for cleanup
- Always follow allocations with defer/errDefer
- Descriptive variables (`top`, `value1`, `operand` not `a`, `b`)
- Logging: use `log.zig` (`log.debug`, `log.warn`)
- Assertions: `tracer.assert(condition, "message")`
- Stack semantics: LIFO order (first pop = top)

### Memory Management

```zig
// Pattern 1: Same scope
const thing = try allocator.create(Thing);
defer allocator.destroy(thing);

// Pattern 2: Ownership transfer
const thing = try allocator.create(Thing);
errdefer allocator.destroy(thing);
thing.* = try Thing.init(allocator);
return thing;
```

### ArrayList API (Zig 0.15.1)

**CRITICAL**: In Zig 0.15.1, `std.ArrayList(T)` returns an UNMANAGED type that requires allocator for all operations!

```zig
// CORRECT: std.ArrayList is UNMANAGED (no internal allocator)
var list = std.ArrayList(T){};  // Default initialization
// OR
const list = std.ArrayList(T).empty;  // Empty constant
// OR with capacity
var list = try std.ArrayList(T).initCapacity(allocator, 100);

// All operations REQUIRE allocator:
defer list.deinit(allocator);  // ✅ allocator REQUIRED
try list.append(allocator, item);  // ✅ allocator REQUIRED
try list.ensureCapacity(allocator, 100);  // ✅ allocator REQUIRED
_ = list.pop();  // No allocator needed for pop

// Direct access (no allocator needed):
list.items[0] = value;
list.items.len = 0;

// WRONG - This does NOT work in Zig 0.15.1:
var list = std.ArrayList(T).init(allocator);  // ❌ No init() method!
list.deinit();  // ❌ Missing required allocator
try list.append(item);  // ❌ Missing required allocator

// For managed ArrayList with internal allocator, use array_list module directly:
const array_list = @import("std").array_list;
var list = array_list.AlignedManaged(T, null).init(allocator);
defer list.deinit();  // No allocator needed for managed version
```

## Testing Philosophy

- NO abstractions - copy/paste setup
- NO helpers - self-contained tests
- Test failures = fix immediately
- Evidence-based debugging only
- **CRITICAL**: Zig tests output NOTHING when passing (no output = success)
- If tests produce no output, they PASSED successfully
- Only failed tests produce output

### Debug Logging in Tests

Enable with:

```zig
test {
    std.testing.log_level = .debug;
}
```

**IMPORTANT**: Even with `std.testing.log_level = .debug`, if the test passes, you will see NO OUTPUT. This is normal Zig behavior. No output means the test passed.

## Project Architecture

### Guillotine EVM

High-performance EVM: correctness, minimal allocations, strong typing.

### Module System

Use `zig build test` not `zig test`. Common error: "primitives" package requires module system.

### Key Components

**Core**: evm.zig, frame.zig, stack.zig, memory.zig, dispatch.zig
**Handlers**: handlers*\*.zig (arithmetic, bitwise, comparison, context, jump, keccak, log, memory, stack, storage, system)
**Synthetic**: handlers*\*\_synthetic.zig (fused ops)
**State**: database.zig, journal.zig, access_list.zig, memory_database.zig
**External**: precompiles.zig, call_params.zig, call_result.zig
**Bytecode**: bytecode.zig, bytecode_analyze.zig, bytecode_stats.zig
**Infrastructure**: tracer.zig, hardfork.zig, eips.zig
**Tracer**: MinimalEvm.zig (65KB standalone), pc_tracker.zig, MinimalEvm_c.zig (WASM FFI)

### Import Rules

```zig
// Good
const Evm = @import("evm");
const memory = @import("memory.zig");

// Bad - no parent imports
const Contract = @import("../frame/contract.zig");
```

## Core Protocols

### Working Directory

**CRITICAL: ALWAYS work from /Users/williamcory/tevm (the monorepo root), NOT the guillotine subdirectory.**

All paths must be relative to the monorepo root:
- `guillotine/src/instructions/Stack.zig` (correct)
- `guillotine/src/instructions/Stack.test.zig` (correct)
- NOT `src/instructions/Stack.zig` (wrong - missing guillotine/ prefix)

### Directory Structure Rules

**NEVER create new directories.** Use existing directory structure:
- `guillotine/src/instructions/` - Instruction implementations (existing)
- Tests go in same directory with `.test.zig` suffix
- NO subdirectories like `guillotine/src/instructions/shared/`
- NO new nested structures

## Commands

### Basic Commands

- `zig build` - Build the project
- `zig build test` - Run all tests (specs → integration → unit)
- `zig build specs` - Run Ethereum execution spec tests
- `zig build test-integration` - Run integration tests from test/\*_/_.zig
- `zig build test-unit` - Run unit tests from src/\*_/_.zig
- `zig build test-lib` - Run library tests from lib/\*_/_.zig
- `zig build test-opcodes` - Run opcode differential tests

### Test Organization

**Test Categories:**

1. **Specs Tests** (`zig build specs`) - Ethereum execution spec compliance tests
2. **Integration Tests** (`zig build test-integration`) - Cross-module testing, differential testing, fixtures
3. **Unit Tests** (`zig build test-unit`) - Module-specific unit tests from src/
4. **Library Tests** (`zig build test-lib`) - External library wrapper tests

**Test Aggregator Files:**

- `src/root.zig` - Aggregates all unit tests from src/\*_/_.zig
- `test/root.zig` - Aggregates all integration tests from test/\*_/_.zig
- `lib/root.zig` - Aggregates all library tests from lib/\*_/_.zig
- `test/specs/ethereum_specs_test.zig` - Ethereum spec test runner

### Test Filtering

Use `-Dtest-filter='<pattern>'` to run specific tests:

```bash
# Run specific test by name
zig build test-opcodes -Dtest-filter='ADD opcode'

# Run tests matching a pattern
zig build test-integration -Dtest-filter='trace validation'

# Filter unit tests
zig build test-unit -Dtest-filter='stack'
```

### Other Test Commands

- `zig build test-snailtracer` - Snailtracer differential test
- `zig build test-synthetic` - Synthetic opcode tests
- `zig build test-fusions` - Fusion optimization tests

## EVM Architecture

### CRITICAL: Dispatch-Based Execution Model

**Guillotine uses a dispatch-based execution model, NOT a traditional interpreter!**

#### Traditional Interpreter (MinimalEvm)

```
Bytecode: [0x60, 0x01, 0x60, 0x02, 0x01, 0x56, 0x5b, 0x00]
           PUSH1  1   PUSH1  2   ADD  JUMP JUMPDEST STOP

Execution: while (pc < bytecode.len) {
    opcode = bytecode[pc]
    switch(opcode) { ... }  // Big switch statement
    pc++
}
```

#### Dispatch-Based Execution (Frame)

```
Bytecode: [0x60, 0x01, 0x60, 0x02, 0x01, 0x56, 0x5b, 0x00]

Dispatch Schedule (preprocessed):
[0] = first_block_gas { gas: 15 }     // Metadata for basic block
[1] = &push_handler                   // Function pointer
[2] = push_inline { value: 1 }        // Inline metadata
[3] = &push_handler                   // Function pointer
[4] = push_inline { value: 2 }        // Inline metadata
[5] = &add_handler                    // Function pointer
[6] = &jump_handler                   // Function pointer
[7] = &jumpdest_handler               // Function pointer
[8] = jump_dest { gas: 3, min: 0 }    // Gas for next block
[9] = &stop_handler                   // Function pointer

Execution: cursor[0].opcode_handler(frame, cursor) → tail calls
```

**Key Differences:**

1. **No PC in Frame**: Frame uses cursor (pointer into dispatch schedule)
2. **No Switch Statement**: Direct function pointer calls with tail-call optimization
3. **Preprocessed**: Bytecode analyzed once, schedule reused
4. **Inline Metadata**: Data embedded directly in schedule (no bytecode reads)
5. **Gas Batching**: Gas calculated per basic block, not per instruction

**Schedule Index ≠ PC**: Schedule[0] might be metadata, not the PC=0 instruction!

### Design Patterns

1. Strong error types per component
2. Unsafe ops for performance (pre-validated)
3. Cache-conscious struct layout
4. Handler tables for O(1) dispatch
5. Bytecode optimization via Dispatch

### Key Separations

- **Frame**: Executes dispatch schedule (NOT bytecode)
- **Dispatch**: Builds optimized schedule from bytecode
- **Host**: External operations

### Opcode Pattern

```zig
pub fn add(self: *Self, cursor: [*]const Dispatch.Item) Error!noreturn {
    self.beforeInstruction(.ADD, cursor);
    self.getTracer().assert(self.stack.size() >= 2, "ADD requires 2 stack items");
    const b = self.stack.pop_unsafe();  // Top of stack
    const a = self.stack.peek_unsafe(); // Second item
    self.stack.set_top_unsafe(a +% b);
    const op_data = dispatch.getOpData(.ADD);
    self.afterInstruction(.ADD, op_data.next_handler, op_data.next_cursor.cursor);
    return @call(Self.getTailCallModifier(), op_data.next_handler, .{ self, op_data.next_cursor.cursor });
}
```

## Opcode Navigation

Handlers organized by type:

- Arithmetic: `handlers_arithmetic.zig`
- Stack: `handlers_stack.zig`
- Memory: `handlers_memory.zig`
- System: `handlers_system.zig`

## Recent Updates

### Tracer System

- Replaced `std.debug.assert` with `tracer.assert()`
- Bytecode analysis lifecycle tracking
- Cursor-aware dispatch sync
- Fixed MinimalEvm stack semantics (LIFO)

### WASM Integration

- C FFI wrapper (MinimalEvm_c.zig)
- Opaque handle pattern
- Complete EVM lifecycle in WASM

### Dispatch Optimization

- Static jump resolution
- Dispatch cache
- Fusion detection
- 300M instruction safety limit

### Memory Management

- Checkpoint system
- Lazy word-aligned allocation
- Cached gas calculations
- Borrowed vs owned memory

## Tracer System Architecture: Execution Synchronization

The tracer system in `@src/tracer/tracer.zig` provides sophisticated execution synchronization between Frame (optimized) and MinimalEvm (reference) implementations:

### How Synchronization Works

**Frame executes a dispatch schedule, MinimalEvm executes bytecode sequentially.**

**Every instruction handler MUST call `self.beforeInstruction(opcode, cursor)`** which:

1. Executes the equivalent operation(s) in MinimalEvm
2. For regular opcodes: Execute 1 MinimalEvm step
3. For synthetic opcodes: Execute N MinimalEvm steps (where N = number of fused operations)
4. Validates that both implementations reach identical state

**CRITICAL**: Frame's cursor is an index into the dispatch schedule, NOT a PC!

- Schedule[0] might be `first_block_gas` metadata, not PC=0
- Schedule indices do NOT correspond to bytecode PCs
- Synthetic handlers in schedule represent multiple bytecode operations

### Instruction Handler Pattern

```zig
pub fn some_opcode(self: *FrameType, cursor: [*]const Dispatch.Item) Error!noreturn {
    self.beforeInstruction(.SOME_OPCODE, cursor);  // ← REQUIRED!
    // ... opcode implementation ...
    return next_instruction(self, cursor, .SOME_OPCODE);
}
```

**CRITICAL**: Missing `beforeInstruction()` calls cause test failures because MinimalEvm gets out of sync.

### Synthetic Opcode Handling

The tracer automatically handles synthetic opcodes in `executeMinimalEvmForOpcode()`:

- **Regular opcodes**: Execute exactly 1 MinimalEvm step
- **PUSH_MSTORE_INLINE**: Execute 2 steps (PUSH1 + MSTORE)
- **FUNCTION_DISPATCH**: Execute 4 steps (PUSH4 + EQ + PUSH + JUMPI)
- **etc.**

This is NOT a divergence issue - it's the designed synchronization mechanism.

### Common Test Failure Root Causes

1. **Dispatch Schedule Misalignment** - Schedule[0] contains metadata, not PC=0 handler
2. **Missing beforeInstruction() calls** - Handler doesn't synchronize MinimalEvm
3. **MinimalEvm context mismatch** - Hardcoded values don't match Frame's blockchain context
4. **Implementation bugs** - Logic errors in either Frame or MinimalEvm

**Key Debugging Points:**

- Frame cursor != PC (cursor is dispatch schedule index)
- Schedule may start with metadata items (first_block_gas)
- Synthetic opcodes in Frame = multiple steps in MinimalEvm
- The tracer's `executeMinimalEvmForOpcode()` handles fusion → sequential mapping correctly

## References

- Zig docs: https://ziglang.org/documentation/0.15.1/
- revm/: Reference Rust implementation
- Yellow Paper: Ethereum spec
- EIPs: Ethereum Improvement Proposals

## Collaboration

- Present proposals, wait for approval
- Plan fails: STOP, explain, wait for guidance

## GitHub Issue Management

Always disclose Claude AI assistant actions:
"_Note: This action was performed by Claude AI assistant, not @roninjin10 or @fucory_"

Required for: creating, commenting, closing, updating issues and all GitHub API operations.

## Build Commands

Usage: `zig build [steps] [options]`

Key Steps:
test Run all tests (specs -> integration -> unit)
specs Run Ethereum execution spec tests
test-integration Run integration tests from test/**/\*.zig
test-unit Run unit tests from src/**/_.zig
test-lib Run library tests from lib/\*\*/_.zig
test-opcodes Run all per-opcode differential tests
test-snailtracer Run snailtracer differential test
test-synthetic Test synthetic opcodes
test-fixtures-differential Run differential tests
test-fusions Run focused fusion tests (unit + dispatch + differential)
wasm Build WASM library and show bundle size
wasm-minimal-evm Build MinimalEvm WASM and show bundle size
wasm-debug Build debug WASM for analysis
python Build Python bindings
swift Build Swift bindings
go Build Go bindings
ts Build TypeScript bindings

Options:
--release[=mode] Release mode: fast, safe, small
-Doptimize=[enum] Debug, ReleaseSafe, ReleaseFast, ReleaseSmall
-Dtest-filter=[string] Filter tests by pattern (e.g., -Dtest-filter='ADD opcode')
-Devm-hardfork=[string] FRONTIER, HOMESTEAD, BYZANTIUM, BERLIN, LONDON, SHANGHAI, CANCUN (default: CANCUN)
-Devm-disable-gas=[bool] Disable gas checks (testing only)
-Devm-enable-fusion=[bool] Enable bytecode fusion (default: true)
-Devm-optimize=[string] EVM optimization strategy: fast, small, or safe (default: safe)
-Dno_precompiles=[bool] Disable all EVM precompiles for minimal build
</target-repo>
</context>

  <source-priority>
    guillotine-mini is the production-ready, spec-compliant EVM. When logic or tests differ between implementations, use guillotine-mini as source of truth. It passes official Ethereum execution specs.
  </source-priority>

  <implementation-context>
    <guillotine-mini>
      Simple, naive implementation. Each opcode handler directly operates on Frame, consuming static gas and incrementing PC. Located in guillotine-mini/src/instructions/handlers_*.zig.
    </guillotine-mini>
    <guillotine>
      Performance-optimized with dispatch-based execution. Handlers use beforeInstruction/afterInstruction hooks for tracing (compiled out in release). Bytecode preprocessed into dispatch schedule (64-byte items: function pointers + metadata). Static gas batched at jump destinations. Includes synthetic fused opcodes. Located in guillotine/src/instructions/handlers_*.zig.
    </guillotine>
    <new-approach>
      Build shared instruction implementations in guillotine/src/instructions/. Generic over FrameType. No static gas charging (caller's responsibility). No PC manipulation. Pure stack operations via frame.stack interface. Frame decides safe/unsafe operations.
    </new-approach>
  </implementation-context>

  <phases>
    <phase id="1">
      <name>Stack-Only Instructions (TDD)</name>
      <goal>Create shared instruction implementations for all stack-only opcodes (~74 opcodes) using Test-Driven Development</goal>

      <instruction-pattern>
        <generic-structure>
          pub fn OpInstruction(comptime FrameType: type) type {
              return struct {
                  pub fn run(frame: *FrameType) FrameType.Error!void {
                      // guillotine-mini logic (source of truth)
                      // No static gas charging
                      // No PC manipulation
                      // Optimize with peek/set_top patterns
                  }
              };
          }
        </generic-structure>
        <required-frame-interface>
          // Safe operations (with error handling)
          frame.stack.pop() -> Error!u256
          frame.stack.push(value: u256) -> Error!void
          frame.stack.peek() -> Error!u256
          frame.stack.set_top(value: u256) -> Error!void
          frame.stack.dup_n(comptime n: u8) -> Error!void   // n = 1..16
          frame.stack.swap_n(comptime n: u8) -> Error!void  // n = 1..16

          Note: guillotine uses pointer-based downward stack with _unsafe variants
          Note: guillotine-mini uses ArrayList
          Note: Both can support this minimal interface
        </required-frame-interface>
        <error-handling>
          Use FrameType.Error pattern throughout
          Phase 1 uses guillotine-mini error set as baseline:
          error{ StackOverflow, StackUnderflow, OutOfGas, OutOfBounds, ... }
        </error-handling>
      </instruction-pattern>

      <deliverables>
        guillotine/src/instructions/Frame.zig - Minimal real frame (grows with needs)
        guillotine/src/instructions/Stack.zig - Stack implementation
        guillotine/src/instructions/instruction.zig - Common types/patterns
        guillotine/src/instructions/arithmetic.zig + arithmetic.test.zig - 11 opcodes
        guillotine/src/instructions/bitwise.zig + bitwise.test.zig - 9 opcodes
        guillotine/src/instructions/comparison.zig + comparison.test.zig - 6 opcodes
        guillotine/src/instructions/stack.zig + stack.test.zig - 48 opcodes

        Pattern: Every .zig file has corresponding .test.zig for unit tests
      </deliverables>

      <sub-phases>
        <sub-phase id="1.1">
          <name>Foundation (TDD)</name>
          <test-first>
            Write failing tests for minimal Frame with Stack
            Test stack interface: pop(), push(), peek(), set_top()
            Test error handling and bounds checking
          </test-first>
          <implement>
            Create Frame.zig with stack field and Error type
            Create Stack.zig (reference guillotine/src/stack/stack.zig for patterns)
            Implement basic operations with proper error handling
          </implement>
          <validate>
            All tests pass
            zig build test-unit runs successfully
          </validate>
        </sub-phase>

        <sub-phase id="1.2">
          <name>Arithmetic Instructions (TDD - 11 opcodes)</name>
          <opcodes>ADD MUL SUB DIV SDIV MOD SMOD ADDMOD MULMOD EXP SIGNEXTEND</opcodes>

          <critical-instruction>
            ⚠️ GUILLOTINE-MINI IS ALWAYS THE SOURCE OF TRUTH ⚠️

            Source: @guillotine-mini/src/instructions/handlers_arithmetic.zig
            Reference: guillotine-mini passes all ethereum/tests - trust its implementation

            If tests fail: YOU copied incorrectly, NOT guillotine-mini being wrong.
          </critical-instruction>

          <tdd-cycle per-opcode="true">
            1. READ the reference implementation from guillotine-mini FIRST
               File: @guillotine-mini/src/instructions/handlers_arithmetic.zig

            2. Write tests by COPYING test patterns from guillotine-mini
               Look at existing tests, copy expected behavior verbatim

            3. COPY the implementation (use cp if helpful as reference):
               ```bash
               # Optional: Create reference copy to compare
               cp guillotine-mini/src/instructions/handlers_arithmetic.zig \
                  guillotine/src/instructions/arithmetic.zig.reference
               ```

            4. Transform MINIMALLY - only these changes:
               a) Wrap in: pub fn AddInstruction(comptime FrameType: type) type { return struct { pub fn run(...) } }
               b) Remove ONLY: try frame.consumeGas(...) lines
               c) Remove ONLY: frame.pc += 1 lines
               d) Change: frame.popStack() → frame.stack.pop()
               e) Change: frame.pushStack(x) → frame.stack.push(x)
               f) Change: frame.peekStack(n) → frame.stack.peek() (if used)

            5. KEEP EVERYTHING ELSE IDENTICAL:
               - Variable names (if guillotine-mini uses 'a' and 'b', you use 'a' and 'b')
               - Operation order (first pop is first pop, second pop is second pop)
               - Logic flow (if statements, calculations, all identical)
               - Comments (copy them too if helpful)

            6. Run tests - if they fail, compare with guillotine-mini line by line

            7. Verify: No static gas charging, no PC manipulation, logic identical to source
          </tdd-cycle>

          <example-transformation>
            SOURCE (@guillotine-mini/src/instructions/handlers_arithmetic.zig):
              pub fn add(frame: *FrameType) FrameType.EvmError!void {
                  try frame.consumeGas(GasConstants.GasFastestStep);  // ← REMOVE
                  const a = try frame.popStack();                      // ← KEEP variable name 'a'
                  const b = try frame.popStack();                      // ← KEEP variable name 'b'
                  try frame.pushStack(a +% b);                         // ← KEEP operation 'a +% b'
                  frame.pc += 1;                                       // ← REMOVE
              }

            TARGET (guillotine/src/instructions/arithmetic.zig):
              pub fn AddInstruction(comptime FrameType: type) type {
                  return struct {
                      pub fn run(frame: *FrameType) FrameType.Error!void {
                          const a = try frame.stack.pop();     // ← Same variable name
                          const b = try frame.stack.pop();     // ← Same variable name
                          try frame.stack.push(a +% b);        // ← Same operation
                          // No gas, no PC - caller's responsibility
                      }
                  };
              }

            DO NOT rename to 'top'/'second' or change operation order.
            DO NOT "improve" or "optimize" the logic.
            TRUST guillotine-mini's implementation completely.
          </example-transformation>
        </sub-phase>

        <sub-phase id="1.3">
          <name>Bitwise Instructions (TDD - 9 opcodes)</name>
          <opcodes>AND OR XOR NOT BYTE SHL SHR SAR</opcodes>

          <critical-instruction>
            ⚠️ GUILLOTINE-MINI IS THE SOURCE OF TRUTH ⚠️
            Source: @guillotine-mini/src/instructions/handlers_bitwise.zig
            Process: Same TDD cycle as Phase 1.2 - COPY, don't rewrite
          </critical-instruction>
        </sub-phase>

        <sub-phase id="1.4">
          <name>Comparison Instructions (TDD - 6 opcodes)</name>
          <opcodes>LT GT SLT SGT EQ ISZERO</opcodes>

          <critical-instruction>
            ⚠️ GUILLOTINE-MINI IS THE SOURCE OF TRUTH ⚠️
            Source: @guillotine-mini/src/instructions/handlers_comparison.zig
            Process: Same TDD cycle as Phase 1.2 - COPY, don't rewrite
          </critical-instruction>
        </sub-phase>

        <sub-phase id="1.5">
          <name>Stack Manipulation (TDD - 48 opcodes)</name>
          <opcodes>
            POP (1)
            PUSH1-PUSH32 (32)
            DUP1-DUP16 (16)
            SWAP1-SWAP16 (16)
          </opcodes>

          <critical-instruction>
            ⚠️ GUILLOTINE-MINI IS THE SOURCE OF TRUTH ⚠️
            Source: @guillotine-mini/src/instructions/handlers_stack.zig

            Stack extension first (dup_n/swap_n), then COPY handler implementations.
            DO NOT invent your own DUP/SWAP logic - copy from guillotine-mini exactly.
          </critical-instruction>

          <stack-extension>
            First extend Stack.zig with dup_n and swap_n (TDD):
              test "dup_n(1) duplicates top" - DUP1 behavior
              test "dup_n(16) duplicates 16th" - DUP16 behavior
              test "swap_n(1) swaps top two" - SWAP1 behavior
              test "swap_n(16) swaps with 17th" - SWAP16 behavior
            Implement:
              pub fn dup_n(self: *Self, comptime n: u8) Error!void
              pub fn swap_n(self: *Self, comptime n: u8) Error!void
          </stack-extension>
        </sub-phase>

        <sub-phase id="1.6">
          <name>Integration Validation</name>
          <cross-category-tests>
            Test sequences: Arithmetic → Bitwise → Comparison
            Test stack manipulation with operations
            Test error propagation across instruction calls
            Test Frame.Error handling
          </cross-category-tests>
          <verification>
            No static gas in any instruction
            No PC manipulation in any instruction
            All zig build test-unit pass
            Frame.zig remains minimal (only stack + errors so far)
          </verification>
        </sub-phase>
      </sub-phases>

      <success-criteria>
        74+ instruction implementations (11 arithmetic + 9 bitwise + 6 comparison + 48 stack)
        All tests pass
        Zero static gas charging in instructions
        Zero PC manipulation in instructions
        Frame.zig minimal but real (foundation for future phases)
        Instructions generic over FrameType
        Stack interface proven via tests
      </success-criteria>
    </phase>

    <phase id="2">
      <name>Context Instructions</name>
      <goal>Instructions that read execution context (no EVM state access)</goal>
      <status>PLACEHOLDER - Design after Phase 1</status>

      <deliverables>
        guillotine/src/instructions/context.zig + context.test.zig
      </deliverables>

      <opcodes count="9">
        ADDRESS, BALANCE (read-only), ORIGIN, CALLER, CALLVALUE
        CALLDATALOAD, CALLDATASIZE, CALLDATACOPY
        CODESIZE, CODECOPY
      </opcodes>

      <required-frame-interface>
        frame.address -> Address
        frame.caller -> Address
        frame.value -> u256
        frame.calldata -> []const u8
        frame.code -> []const u8
      </required-frame-interface>

      <pattern>Same as Phase 1 - strip gas, strip PC, pure logic</pattern>
    </phase>

    <phase id="3">
      <name>Memory Instructions</name>
      <goal>Instructions that operate on EVM memory</goal>
      <status>PLACEHOLDER - Design after Phase 2</status>

      <deliverables>
        guillotine/src/instructions/memory.zig + memory.test.zig
      </deliverables>

      <opcodes count="6">
        MLOAD, MSTORE, MSTORE8, MSIZE, MCOPY (Cancun+)
      </opcodes>

      <required-frame-interface>
        frame.memory -> Memory type
        // Memory interface TBD based on guillotine/src/memory/memory.zig patterns
      </required-frame-interface>

      <key-considerations>
        - Memory expansion gas calculation (dynamic) - KEEP in instruction
        - Quadratic cost formula: size_in_words ** 2 // 512
        - Word-aligned size tracking
        - Hardfork guards (MCOPY only in Cancun+)
      </key-considerations>
    </phase>

    <phase id="4">
      <name>Storage Instructions</name>
      <goal>Instructions that access persistent storage</goal>
      <status>PLACEHOLDER - Design after Phase 3</status>

      <deliverables>
        guillotine/src/instructions/storage.zig + storage.test.zig
      </deliverables>

      <opcodes count="4">
        SLOAD, SSTORE, TLOAD (Cancun+), TSTORE (Cancun+)
      </opcodes>

      <required-frame-interface>
        frame.getEvm() -> *Evm  // Access to storage, warm/cold tracking
        frame.is_static -> bool  // For SSTORE/TSTORE validation
      </required-frame-interface>

      <key-considerations>
        - Dynamic gas costs (warm/cold) - KEEP in instruction
        - Static call violation checks
        - SSTORE sentry gas check (2300)
        - Transient storage (Cancun+): always warm, cleared per-transaction
        - Refund logic (complex, hardfork-dependent)
        - EIP-2929 warm/cold tracking
        - EIP-2200/EIP-3529 storage gas metering
      </key-considerations>
    </phase>

    <phase id="5">
      <name>Log Instructions</name>
      <goal>Event logging instructions</goal>
      <status>PLACEHOLDER - Design after Phase 4</status>

      <deliverables>
        guillotine/src/instructions/log.zig + log.test.zig
      </deliverables>

      <opcodes count="5">
        LOG0, LOG1, LOG2, LOG3, LOG4
      </opcodes>

      <required-frame-interface>
        frame.getEvm() -> *Evm  // Access to log buffer
        frame.is_static -> bool  // Static call violation check
      </required-frame-interface>

      <key-considerations>
        - Dynamic gas costs (data size, topic count) - KEEP in instruction
        - Memory expansion gas calculation
        - Static call violation (cannot LOG in static context)
        - Topic extraction and storage
      </key-considerations>
    </phase>

    <phase id="6">
      <name>System Instructions</name>
      <goal>Complex system operations (CALL, CREATE, SELFDESTRUCT)</goal>
      <status>PLACEHOLDER - Design after Phase 5</status>

      <deliverables>
        guillotine/src/instructions/system.zig + system.test.zig
      </deliverables>

      <opcodes count="9+">
        CALL, CALLCODE, RETURN, DELEGATECALL
        CREATE, CREATE2
        STATICCALL, REVERT, SELFDESTRUCT
        RETURNDATACOPY, RETURNDATASIZE
      </opcodes>

      <required-frame-interface>
        frame.getEvm() -> *Evm
        frame.return_data -> []const u8
        frame.gas_remaining -> i64
        frame.allocator -> Allocator
        // Complex call/create parameter handling TBD
      </required-frame-interface>

      <key-considerations>
        - Extremely complex gas logic
        - EIP-150 (63/64 rule for gas forwarding)
        - Call depth limits (1024 max)
        - Value transfers and balance checks
        - Precompile handling
        - Return data management
        - SELFDESTRUCT only in same tx (EIP-6780 Cancun+)
        - Static call propagation
      </key-considerations>

      <warning>
        This phase may need significant design work due to complexity.
        System instructions interact deeply with EVM orchestration.
      </warning>
    </phase>

    <phase id="7">
      <name>Bytecode Module</name>
      <goal>Unified bytecode analysis and validation</goal>
      <status>PLACEHOLDER - Design after instruction phases</status>

      <key-topics>
        - Bytecode validation
        - Jump destination analysis (JUMPDEST marking)
        - Invalid opcode detection
        - Bytecode size limits (hardfork-dependent)
        - PUSH immediate value extraction
        - Bytecode iteration patterns
      </key-topics>

      <rationale>
        Bytecode analysis is foundation for dispatch system.
        Design after understanding instruction requirements.
      </rationale>
    </phase>

    <phase id="8">
      <name>Dispatch/Jump Instructions</name>
      <goal>Control flow instructions (intentionally deferred)</goal>
      <status>PLACEHOLDER - Requires dispatch architecture understanding</status>

      <opcodes count="6">
        JUMP, JUMPI, JUMPDEST, PC, GAS, STOP
      </opcodes>

      <rationale>
        Jump instructions are tightly coupled to execution model:
        - Traditional: PC-based with bytecode iteration
        - Dispatch: Cursor-based with preprocessed schedule
        Design after bytecode phase to understand both models.
      </rationale>

      <key-considerations>
        - JUMP destination validation
        - Dynamic vs static jump resolution
        - PC tracking vs cursor dispatch
        - Gas batching at jump destinations (dispatch mode)
      </key-considerations>
    </phase>

    <phase id="9">
      <name>EIPs and Hardfork Module</name>
      <goal>Unified hardfork management</goal>
      <status>PLACEHOLDER</status>

      <key-topics>
        - Hardfork enum and detection
        - EIP activation logic
        - Feature flags (isAtLeast, isBefore)
        - Gas constant selection (hardfork-dependent)
        - Opcode availability (e.g., PUSH0 only Shanghai+)
        - Precompile activation
      </key-topics>

      <supported-hardforks>
        FRONTIER, HOMESTEAD, TANGERINE_WHISTLE, SPURIOUS_DRAGON
        BYZANTIUM, CONSTANTINOPLE, PETERSBURG, ISTANBUL
        BERLIN, LONDON, MERGE, SHANGHAI, CANCUN, PRAGUE
      </supported-hardforks>
    </phase>

    <phase id="10">
      <name>Tracing Module</name>
      <goal>Unified EIP-3155 trace support</goal>
      <status>PLACEHOLDER</status>

      <key-topics>
        - Trace capture (PC, opcode, gas, stack, memory)
        - State comparison between executions
        - Divergence analysis and reporting
        - MinimalEvm synchronization (guillotine-specific)
        - beforeInstruction/afterInstruction hooks
      </key-topics>

      <note>
        guillotine has sophisticated tracer that syncs Frame (dispatch) with MinimalEvm (traditional).
        Design must support both tracing modes.
      </note>
    </phase>

    <phase id="11">
      <name>Frame Module</name>
      <goal>Unified Frame implementation supporting multiple execution modes</goal>
      <status>PLACEHOLDER</status>

      <key-topics>
        - Traditional interpreter mode (PC-based sequential execution)
        - Dispatch mode (cursor-based with preprocessed schedule)
        - Mode selection (compile-time or runtime)
        - Shared state management (stack, memory, gas)
        - Gas tracking and refunds
        - Call context management
      </key-topics>

      <modes>
        Mode 1 (Traditional): PC loop, bytecode read, switch dispatch
        Mode 2 (Dispatch): Cursor pointer, schedule read, tail calls
      </modes>
    </phase>

    <phase id="12">
      <name>Evm Module</name>
      <goal>Unified EVM orchestrator</goal>
      <status>PLACEHOLDER</status>

      <key-topics>
        - State management (accounts, balances, nonces, code)
        - Storage (persistent + transient)
        - Warm/cold access tracking (EIP-2929)
        - Gas refunds (capped 1/5 in London+)
        - Nested call handling (CALL/CREATE)
        - Call depth management (1024 max)
        - Host interface for external state
      </key-topics>

      <critical-components>
        - inner_call() - Nested CALL/STATICCALL/DELEGATECALL
        - inner_create() - Nested CREATE/CREATE2
        - accessAddress() - EIP-2929 address tracking
        - accessStorageSlot() - EIP-2929 storage tracking
      </critical-components>
    </phase>

    <phase id="13">
      <name>End-to-End Tests</name>
      <goal>Comprehensive integration testing</goal>
      <status>PLACEHOLDER</status>

      <key-topics>
        - Ethereum execution-specs tests (ethereum/tests)
        - Cross-mode validation (traditional vs dispatch)
        - Performance benchmarks
        - Differential testing (vs revm, vs geth)
        - Hardfork-specific test suites
        - Edge case coverage
      </key-topics>

      <test-suites>
        - GeneralStateTests (ethereum/tests)
        - VMTests
        - BlockchainTests
        - Differential vs guillotine-mini (source of truth)
      </test-suites>
    </phase>
  </phases>

  <success-criteria>
    Phase 1: 74 instruction implementations, all tests pass, zero gas/PC in instructions
    Overall: Single guillotine implementation supporting multiple interpretation modes
  </success-criteria>
</task>
```

## IMPORTANT UPDATE THIS PROMPT

Update this prompt if you find a change should be made for new agents
