# Merge guillotine-mini into guillotine

Merge @guillotine-mini/ into @guillotine/ submodule.

## Prompt

```xml
<task>
  <objective>Merge guillotine-mini → guillotine</objective>
  
  <context>
    <source>
      ⚠️ GUILLOTINE-MINI = SOURCE OF TRUTH ⚠️

      Docs: @guillotine-mini/CLAUDE.md
      Files: @guillotine-mini/src/instructions/handlers_*.zig

      Passes ethereum/tests. Spec-compliant. Trust it completely.
      Test fails = YOU copied wrong, not source being wrong.
    </source>

    <target>
      Docs: @guillotine/CLAUDE.md, @guillotine/src/instructions/CLAUDE.md

      GOAL: Build single canonical EVM implementation from scratch.
      This REPLACES both guillotine and guillotine-mini with one unified codebase.
      Not a compatibility layer - a clean reimplementation using guillotine-mini as reference.
    </target>
  </context>

  <implementation-context>
    guillotine-mini: Simple PC-based. Handlers: consumeGas() → stack ops → pc++. @guillotine-mini/src/instructions/handlers_*.zig
    guillotine: Dispatch-based. Handlers: beforeInstruction() → stack ops → tail call. @guillotine/src/instructions/handlers_*.zig

    NEW CANONICAL IMPLEMENTATION: @guillotine/src/instructions/
    - Generic over FrameType (works with any Frame that provides required interface)
    - Static gas removed (GasFastestStep, GasQuickStep, etc. - caller handles)
    - Dynamic gas KEPT as frame methods (memoryExpansionCost, accessStorageSlot, etc.)
    - PC removed (caller handles program counter advancement)
    - Instructions focus on business logic: read inputs, perform operation, write outputs
  </implementation-context>

  <gas-handling-rules>
    ⚠️ CRITICAL: Understand static vs dynamic gas ⚠️

    TWO TYPES OF GAS:

    1. STATIC GAS (REMOVE from instructions):
       - Constants: GasFastestStep (3), GasQuickStep (2), GasFastStep (5), GasMidStep (8), etc.
       - Always the same value regardless of inputs/state
       - Caller charges these before/after instruction
       - Example: try frame.consumeGas(GasConstants.GasFastestStep); // ← REMOVE THIS

    2. DYNAMIC GAS (KEEP via frame methods):
       - Memory expansion: frame.memoryExpansionCost(end_bytes) - quadratic cost
       - Storage access: frame.accessStorageSlot(addr, key) - warm/cold tracking (EIP-2929)
       - Account access: frame.accessAddress(addr) - warm/cold tracking
       - Hardfork-dependent: Different costs per fork
       - These methods may consume gas internally OR return cost for instruction to consume
       - Example: const cost = frame.memoryExpansionCost(end); // ← KEEP THIS

    TRANSFORMATION EXAMPLE:
    SOURCE (guillotine-mini):
      try frame.consumeGas(GasConstants.GasFastestStep);  // Static - REMOVE
      const end_bytes: u64 = offset + 32;
      const mem_cost = frame.memoryExpansionCost(end_bytes);  // Dynamic - KEEP
      try frame.consumeGas(mem_cost);  // Might be KEEP or REMOVE (check if method consumes internally)

    TARGET (new canonical):
      // Static gas removed - caller handles GasFastestStep
      const end_bytes: u64 = offset + 32;
      const mem_cost = frame.memoryExpansionCost(end_bytes);  // Keep dynamic calculation
      // Check: Does memoryExpansionCost consume internally? Or return cost to consume?
  </gas-handling-rules>

  <critical-lessons>
    ACTUAL BUGS FROM THIS SESSION:
    1. Renamed a/b → top/second → confused stack order → ALL TESTS FAILED
    2. Didn't copy, rewrote logic → wrong operand order (top-second vs second-top)
    3. Tried to "optimize" → broke correct implementation

    ROOT CAUSE: Ignoring "COPY FROM SOURCE" instruction.
    FIX: Mechanical transformation ONLY. Trust source completely.

    Working directory: ALWAYS /Users/williamcory/tevm (monorepo root), NOT submodules.
    Paths: guillotine/src/... and guillotine-mini/src/... (from root).
    Commands: Use subshells: (cd guillotine && zig build test-unit)

    Zig 0.15.1 ArrayList: UNMANAGED - all ops need allocator:
    - std.ArrayList(T){} - default init
    - list.deinit(allocator) - NOT list.deinit()
    - list.append(allocator, item) - NOT list.append(item)

    Stack semantics: LIFO. First pop = TOP (most recent). Second pop = SECOND.
    Example: push(10), push(3) → stack=[10,3] → pop()=3 (top), pop()=10 (second)
    For SUB: result = second - top = 10 - 3 = 7 (NOT top - second)

    Commit strategy: After each completed sub-phase (1.1, 1.2, etc).
    Use @chop/.claude/commands/commit.md format (emoji, conventional, co-author).
  </critical-lessons>

  <phases>
    <phase id="1">
      <name>Stack-Only Instructions (TDD)</name>
      <goal>~74 opcodes, shared implementations</goal>

      <pattern>
        pub fn OpInstruction(comptime FrameType: type) type {
            return struct {
                pub fn run(frame: *FrameType) FrameType.Error!void {
                    // ⚠️ COPY FROM GUILLOTINE-MINI - MECHANICAL TRANSFORMATION ONLY ⚠️

                    REMOVE:
                    - try frame.consumeGas(GasConstants.StaticConstant)  // All static gas
                    - frame.pc += 1  // All PC manipulation

                    CHANGE:
                    - frame.popStack() → frame.stack.pop()
                    - frame.pushStack(x) → frame.stack.push(x)

                    KEEP UNCHANGED:
                    - Variable names (a, b, offset, value - exactly as in source)
                    - Operation order (don't reorder for "clarity")
                    - Logic (don't "simplify" or "optimize")
                    - Comments (copy verbatim)
                    - Dynamic gas calls (frame.memoryExpansionCost, etc.)
                    - Hardfork checks (if evm.hardfork.isAtLeast(...))
                    - Error conditions (bounds checks, static call violations, etc.)
                }
            };
        }

        Frame interface (Phase 1):
          frame.stack.{pop,push,peek,set_top,dup_n,swap_n}() -> Error!...
          frame.bytecode: []const u8
          frame.pc: u32 (read-only, for PUSH immediate reading)
          frame.readImmediate(size: u8) ?u256

        Errors: {StackOverflow,StackUnderflow,OutOfGas,OutOfBounds,InvalidPush,...}
      </pattern>

      <deliverables>
        @guillotine/src/instructions/{Frame,Stack,arithmetic,bitwise,comparison,stack}.{zig,test.zig}
      </deliverables>

      <sub-phases>
        <sub-phase id="1.1">
          <name>Foundation</name>
          TDD: Frame.zig + Stack.zig. Tests → impl → validate.
        </sub-phase>

        <sub-phase id="1.2">
          <name>Arithmetic (11 ops)</name>
          <ops>ADD MUL SUB DIV SDIV MOD SMOD ADDMOD MULMOD EXP SIGNEXTEND</ops>
          
          ⚠️ SOURCE: @guillotine-mini/src/instructions/handlers_arithmetic.zig ⚠️
          
          Process:
          1. READ guillotine-mini
          2. COPY tests from guillotine-mini
          3. Optional: cp handlers_arithmetic.zig arithmetic.zig.reference
          4. Transform ONLY:
             - Wrap: pub fn AddInstruction(comptime FrameType: type) type { return struct { pub fn run(...) } }
             - Remove: try frame.consumeGas(...)
             - Remove: frame.pc += 1
             - Change: frame.{pop,push}Stack() → frame.stack.{pop,push}()
          5. KEEP IDENTICAL: variable names, operation order, logic, comments
          6. Test fails → compare with source line-by-line
          
          Example:
          SOURCE: const a = try frame.popStack(); const b = try frame.popStack(); try frame.pushStack(a +% b);
          TARGET: const a = try frame.stack.pop(); const b = try frame.stack.pop(); try frame.stack.push(a +% b);
          
          DON'T rename a/b → top/second. DON'T "improve" logic. TRUST source.
        </sub-phase>

        <sub-phase id="1.3">
          <name>Bitwise (9 ops)</name>
          <ops>AND OR XOR NOT BYTE SHL SHR SAR</ops>
          ⚠️ SOURCE: @guillotine-mini/src/instructions/handlers_bitwise.zig ⚠️
          Same TDD as 1.2 - COPY, don't rewrite
        </sub-phase>

        <sub-phase id="1.4">
          <name>Comparison (6 ops)</name>
          <ops>LT GT SLT SGT EQ ISZERO</ops>
          ⚠️ SOURCE: @guillotine-mini/src/instructions/handlers_comparison.zig ⚠️
          Same TDD as 1.2 - COPY, don't rewrite
        </sub-phase>

        <sub-phase id="1.5">
          <name>Stack (48 ops)</name>
          <ops>POP PUSH1-32 DUP1-16 SWAP1-16</ops>
          ⚠️ SOURCE: @guillotine-mini/src/instructions/handlers_stack.zig ⚠️
          Extend Stack.zig: dup_n/swap_n. Then COPY handlers.
        </sub-phase>

        <sub-phase id="1.6">
          <name>Validation</name>
          Test sequences. Verify: no gas/PC in instructions, all tests pass, Frame minimal.
        </sub-phase>
      </sub-phases>

      <success>74+ instructions, tests pass, zero gas/PC, generic over FrameType</success>
    </phase>

    <phase id="2">
      <name>Context Operations (17 ops)</name>
      <goal>Execution context access: addresses, balances, code, calldata, gas</goal>
      <source>@guillotine-mini/src/instructions/handlers_context.zig</source>

      <pattern>
        pub fn OpInstruction(comptime FrameType: type) type {
            return struct {
                pub fn run(frame: *FrameType) FrameType.Error!void {
                    // ⚠️ COPY FROM GUILLOTINE-MINI - MECHANICAL TRANSFORMATION ONLY ⚠️

                    REMOVE:
                    - try frame.consumeGas(GasConstants.StaticConstant)  // All static gas
                    - frame.pc += 1  // All PC manipulation

                    CHANGE:
                    - frame.popStack() → frame.stack.pop()
                    - frame.pushStack(x) → frame.stack.push(x)
                    - frame.getEvm() → keep (delegate to parent EVM)

                    KEEP UNCHANGED:
                    - Variable names (addr_int, off, len - exactly as in source)
                    - Operation order (especially for memory operations)
                    - Logic (don't simplify)
                    - Dynamic gas calls (frame.memoryExpansionCost, evm.accessAddress, etc.)
                    - Hardfork checks (if evm.hardfork.isAtLeast(...))
                    - Error conditions
                }
            };
        }

        Frame interface additions:
          // Fields (from Phase 1.1):
          frame.caller: Address
          frame.address: Address
          frame.value: u256
          frame.calldata: []const u8
          frame.bytecode: Bytecode (or []const u8)
          frame.return_data: []const u8
          frame.gas_remaining: i64
          frame.hardfork: Hardfork

          // Memory methods (from Phase 1.1):
          frame.readMemory(offset: u32) u8
          frame.writeMemory(offset: u32, value: u8) Error!void
          frame.memoryExpansionCost(end_bytes: u64) u64

          // NEW - EVM state access:
          frame.getEvm() *EvmType
            Returns pointer to parent EVM for:
            - evm.origin: Address (transaction originator)
            - evm.gas_price: u256 (transaction gas price)
            - evm.balances: HashMap(Address, u256)
            - evm.code: HashMap(Address, []const u8)
            - evm.nonces: HashMap(Address, u64)
            - evm.hardfork: Hardfork
            - evm.accessAddress(addr: Address) u64  // EIP-2929: returns cold/warm cost
            - evm.host: ?HostInterface (for external account state if available)

        Errors: {StackOverflow,StackUnderflow,OutOfGas,OutOfBounds,InvalidOpcode}
      </pattern>

      <ops>
        ADDRESS BALANCE ORIGIN CALLER CALLVALUE
        CALLDATALOAD CALLDATASIZE CALLDATACOPY
        CODESIZE CODECOPY GASPRICE
        EXTCODESIZE EXTCODECOPY RETURNDATASIZE RETURNDATACOPY EXTCODEHASH
        GAS
      </ops>

      <critical-details>
        1. **EVM Access Pattern**: frame.getEvm() delegates to parent
           - Frame NEVER owns balances/code/storage
           - Just provides typed access to EVM state
           - Example: const evm = frame.getEvm(); const bal = evm.balances.get(addr);

        2. **Hardfork-Aware Gas**:
           - BALANCE: 20 (pre-Tangerine) → 400 (Tangerine-Petersburg) → 700 (Istanbul-Berlin) → 100/2600 (Berlin+)
           - EXTCODESIZE/EXTCODECOPY/EXTCODEHASH: Same progression
           - Use evm.accessAddress() for Berlin+ (handles warm/cold automatically)

        3. **Copy Operations** (CALLDATACOPY, CODECOPY):
           - Base gas + memory expansion + (3 gas × words_copied)
           - Memory expansion charged ONCE for destination range
           - Source data (calldata/code) has no expansion cost

        4. **Return Data** (RETURNDATASIZE, RETURNDATACOPY):
           - Only available Byzantium+: if (evm.hardfork.isBefore(.BYZANTIUM)) return error.InvalidOpcode
           - Bounds check against frame.return_data.len (NOT memory size)
           - Read from frame.return_data, write to frame.memory

        5. **External Code** (EXTCODESIZE, EXTCODECOPY, EXTCODEHASH):
           - Check evm.host first: if (evm.host) |h| use h.getCode()
           - Fallback to evm.code.get(addr) orelse &[_]u8{}
           - EXTCODEHASH: Constantinople+, returns keccak256 or 0 for empty

        6. **CODESIZE/CODECOPY**:
           - Source: frame.bytecode (NOT external code)
           - CODECOPY from frame.bytecode.getOpcode(index) or bytecode[index]
      </critical-details>

      <deliverables>
        @guillotine/src/instructions/context.{zig,test.zig}
      </deliverables>

      <success>17 instructions, tests pass, getEvm() interface defined</success>
    </phase>

    <phase id="3">
      <name>Memory Operations (5 ops)</name>
      <goal>Memory load/store, size query, memory-to-memory copy</goal>
      <source>@guillotine-mini/src/instructions/handlers_memory.zig</source>
      <status>✅ INTERFACE COMPLETE IN PHASE 1.1</status>

      <pattern>
        Memory interface ALREADY implemented in Phase 1.1 (Frame.zig):
          frame.memory: AutoHashMap(u32, u8)  // Sparse byte storage
          frame.memory_size: u32  // Word-aligned size tracker
          frame.readMemory(offset: u32) u8  // Returns 0 for uninitialized
          frame.writeMemory(offset: u32, value: u8) Error!void  // Auto-expands
          frame.memoryExpansionCost(end_bytes: u64) u64  // Quadratic: 3n + n²/512
          frame.allocator: Allocator  // For MCOPY temporary buffer

        NO NEW Frame methods needed!
      </pattern>

      <ops>
        MLOAD MSTORE MSTORE8 MSIZE MCOPY
      </ops>

      <critical-details>
        1. **Memory Expansion**: Quadratic cost = 3×words + words²/512
           - frame.memoryExpansionCost() already implements this
           - Returns INCREMENTAL cost (current → new size)

        2. **Word Alignment**: Memory always expands to 32-byte boundaries
           - Writing byte at offset 31 → memory_size = 32
           - Writing byte at offset 32 → memory_size = 64

        3. **MLOAD/MSTORE**: Big-endian 32-byte words
           - MLOAD: Read 32 bytes, shift into u256
           - MSTORE: Extract bytes from u256, write 32 bytes

        4. **MCOPY** (EIP-5656, Cancun+):
           - Check hardfork: if (evm.hardfork.isBefore(.CANCUN)) return error.InvalidOpcode
           - Stack: dest, src, len
           - Memory expansion for BOTH src and dest ranges
           - Use temporary buffer to handle overlapping regions
           - Zero-length copy: charge gas but no actual copy

        5. **MSIZE**: Returns frame.memory_size (NOT byte count)
           - Word-aligned value (always multiple of 32)
      </critical-details>

      <deliverables>
        @guillotine/src/instructions/memory.{zig,test.zig}
      </deliverables>

      <success>5 instructions, reuse Phase 1.1 memory interface, MCOPY handles overlaps</success>
    </phase>

    <phase id="4">
      <name>Storage Operations (4 ops)</name>
      <goal>Persistent and transient storage access</goal>
      <source>@guillotine-mini/src/instructions/handlers_storage.zig</source>

      <pattern>
        pub fn OpInstruction(comptime FrameType: type) type {
            return struct {
                pub fn run(frame: *FrameType) FrameType.Error!void {
                    // ⚠️ COPY FROM GUILLOTINE-MINI - EXACT GAS LOGIC ⚠️

                    const evm = frame.getEvm();

                    // SLOAD/SSTORE: Complex hardfork-aware gas
                    // TLOAD/TSTORE: Always 100 gas (warm), Cancun+

                    // Static call check for writes
                    if (frame.is_static) return error.StaticCallViolation;
                }
            };
        }

        Frame interface additions:
          // NEW field:
          frame.is_static: bool  // True if STATICCALL context

          // EVM storage access (via getEvm()):
          evm.storage.get(address: Address, key: u256) u256
          evm.storage.set(address: Address, key: u256, value: u256) Error!void
          evm.storage.getOriginal(address: Address, key: u256) u256  // Pre-transaction value
          evm.storage.getTransient(address: Address, key: u256) u256  // EIP-1153
          evm.storage.setTransient(address: Address, key: u256, value: u256) Error!void
          evm.accessStorageSlot(address: Address, key: u256) u64  // EIP-2929
          evm.add_refund(amount: u64) void
          evm.gas_refund: u64

        Errors: Add StaticCallViolation
      </pattern>

      <ops>
        SLOAD SSTORE TLOAD TSTORE
      </ops>

      <critical-details>
        1. **SSTORE Complexity** (Most complex opcode in EVM):
           - Pre-Istanbul: 20000 (set 0→nonzero), 5000 (update), 15000 refund (clear)
           - Istanbul (EIP-2200): Net gas metering with dirty tracking
           - Berlin (EIP-2929): Cold/warm access (2100 cold, 100 warm)
           - London (EIP-3529): Reduced refunds (4800 for clear)

        2. **Three Storage Values**:
           - original_value: Value at transaction start (from evm.storage.getOriginal)
           - current_value: Value before this SSTORE (from evm.storage.get)
           - new_value: Value being written (from stack)

        3. **Istanbul+ Sentry Check**:
           ```zig
           if (evm.hardfork.isAtLeast(.ISTANBUL)) {
               if (frame.gas_remaining <= GasConstants.SstoreSentryGas) {  // 2300
                   return error.OutOfGas;
               }
           }
           ```

        4. **Gas Cost Calculation** (Istanbul+):
           - Cold access first (if slot not warm): +2100
           - Then compare original vs current vs new:
             - original == current != new: SLOAD cost if original==0, else SLOAD - cold
             - original != current: just warm access (100)
           - Refunds: Complex logic based on 3-way comparison

        5. **Transient Storage** (EIP-1153, Cancun+):
           - Always warm (100 gas), no cold access
           - Cleared at transaction boundaries (NOT call boundaries)
           - No gas refunds
           - Must check is_static for TSTORE

        6. **Static Context**:
           - SSTORE: Check is_static AFTER gas charge (Python line 36)
           - TSTORE: Check is_static AFTER gas charge

        7. **Don't Track Warm/Cold Yourself**:
           - evm.accessStorageSlot() handles tracking internally
           - Returns cold cost (2100) or warm cost (100)
      </critical-details>

      <deliverables>
        @guillotine/src/instructions/storage.{zig,test.zig}
      </deliverables>

      <success>4 instructions, SSTORE matches Python exactly, transient storage works</success>
    </phase>

    <phase id="5">
      <name>Log Operations (5 ops)</name>
      <goal>Emit event logs with topics</goal>
      <source>@guillotine-mini/src/instructions/handlers_log.zig</source>

      <pattern>
        pub fn OpInstruction(comptime FrameType: type) type {
            return struct {
                pub fn run(frame: *FrameType, opcode: u8) FrameType.Error!void {
                    // ⚠️ LOG0-LOG4 share implementation, opcode param determines topic count ⚠️

                    if (frame.is_static) return error.StaticCallViolation;

                    const topic_count = opcode - 0xa0;  // 0xa0=LOG0, 0xa1=LOG1, etc.
                    const offset = try frame.stack.pop();
                    const length = try frame.stack.pop();

                    // Gas: base (375) + topic_cost (375×N) + data_cost (8×bytes) + memory_expansion
                    const log_cost = GasConstants.LogGas + (topic_count * GasConstants.LogTopicGas)
                                   + (length_u32 * GasConstants.LogDataGas);

                    // Read topics from stack (0-4 topics)
                    // Read data from memory
                    // Append to evm.logs
                }
            };
        }

        Frame interface (reuse from Phase 4):
          frame.is_static: bool  // Reuse from Phase 4

        EVM log interface (via getEvm()):
          evm.logs: ArrayList(Log)
          evm.logs.append(allocator, log_entry) Error!void
          evm.arena.allocator() Allocator  // For log data/topics

        Log structure:
          Log {
              address: Address,       // frame.address (contract that emitted log)
              topics: []u256,         // 0-4 indexed topics
              data: []u8,             // Arbitrary data from memory
          }
      </pattern>

      <ops>
        LOG0 LOG1 LOG2 LOG3 LOG4
      </ops>

      <critical-details>
        1. **Shared Implementation**: All 5 opcodes use same handler
           - Take opcode byte as parameter
           - topic_count = opcode - 0xa0

        2. **Gas Calculation**:
           - Base: 375 (GasConstants.LogGas)
           - Per topic: 375 (GasConstants.LogTopicGas)
           - Per data byte: 8 (GasConstants.LogDataGas)
           - Plus memory expansion for data range

        3. **Static Call Check**: FIRST operation (EIP-214)
           - if (frame.is_static) return error.StaticCallViolation

        4. **Topic Order**: Stack pops in REVERSE order
           - LOG2: Stack has [offset, length, topic0, topic1]
           - Pop: offset, length, then topic0, topic1
           - Store in array: [topic0, topic1]

        5. **Memory Read**:
           - If length == 0, no data (but still valid log)
           - Read bytes from frame.memory at offset
           - Allocate in evm.arena (persistent until transaction end)

        6. **Log Accumulation**:
           - Append to evm.logs (transaction-wide accumulator)
           - Logs NOT reverted on subcall failure (accumulated in parent)
           - Only transaction revert clears logs

        7. **Address**: Log always uses frame.address (contract emitting)
           - NOT frame.caller (that's the caller of the contract)
      </critical-details>

      <deliverables>
        @guillotine/src/instructions/log.{zig,test.zig}
      </deliverables>

      <success>5 instructions, shared implementation, topics in correct order</success>
    </phase>

    <phase id="6">
      <name>System Operations (9 ops)</name>
      <goal>Calls, contract creation, self-destruct</goal>
      <source>@guillotine-mini/src/instructions/handlers_system.zig</source>
      <warning>⚠️ MOST COMPLEX PHASE - Nested calls, gas forwarding, context propagation ⚠️</warning>

      <pattern>
        pub fn OpInstruction(comptime FrameType: type) type {
            return struct {
                pub fn run(frame: *FrameType) FrameType.Error!void {
                    // ⚠️ SYSTEM OPS = FULL EVM INTEGRATION ⚠️

                    const evm = frame.getEvm();

                    // CALL family: Calculate gas, prepare params, invoke evm.inner_call()
                    // CREATE family: Calculate gas, prepare init code, invoke evm.inner_create()
                    // SELFDESTRUCT: Transfer balance, mark for deletion
                }
            };
        }

        Frame interface additions:
          // NEW fields:
          frame.is_static: bool  // Propagate to child calls
          frame.stopped: bool  // SELFDESTRUCT sets this

          // NEW methods:
          frame.createGasCost(init_code_len: u32) u64  // Dynamic CREATE gas
          frame.create2GasCost(init_code_len: u32) u64  // CREATE2 adds hash cost
          frame.selfdestructGasCost() u64  // Base 5000
          frame.selfdestructRefund() u64  // Pre-London: 24000, London+: 0

        EVM interface (via getEvm()):
          evm.inner_call(params: CallParams) CallResult
            - CALL, CALLCODE, DELEGATECALL, STATICCALL
            - Returns: { success: bool, output: []u8, gas_left: u64 }

          evm.inner_create(value: u256, init_code: []const u8, gas: u64, salt: ?u256) CallResult
            - CREATE, CREATE2 (salt for CREATE2)
            - Returns: { success: bool, address: Address, output: []u8, gas_left: u64 }

          evm.setBalanceWithSnapshot(address: Address, balance: u256) Error!void
          evm.selfdestructed_accounts: HashMap(Address, void)
          evm.created_accounts: HashMap(Address, void)  // EIP-6780 tracking
          evm.accessAddress(address: Address) u64  // Warm/cold cost

        CallParams variants:
          .call = { caller, to, value, input, gas }
          .callcode = { caller, to, value, input, gas }
          .delegatecall = { caller, to, input, gas }  // No value
          .staticcall = { caller, to, input, gas }  // No value, sets is_static
      </pattern>

      <ops>
        CREATE CALL CALLCODE DELEGATECALL STATICCALL CREATE2 SELFDESTRUCT
      </ops>

      <critical-details>
        1. **Gas Forwarding** (EIP-150, Tangerine Whistle+):
           - All but 1/64th: available_gas = remaining - (remaining / 64)
           - Pre-Tangerine: Forward all remaining gas
           - Calculation BEFORE charging operation cost

        2. **Gas Stipend** (Value Transfers):
           - CALL/CALLCODE with value > 0: Add 2300 gas stipend
           - Stipend is FREE (not charged to caller)
           - Child receives: available_gas + stipend

        3. **Call Gas Components**:
           - Base cost: 100 (warm) or 2600 (cold) - Berlin+
           - Value transfer: 9000 if value > 0
           - New account: 25000 if value > 0 AND account doesn't exist
           - Memory expansion: For both input and output regions

        4. **Call Context**:
           - CALL: New context (msg.sender = caller, msg.value = value)
           - CALLCODE: Caller's context (msg.sender = original, storage = caller's)
           - DELEGATECALL: Delegate context (msg.sender/value preserved)
           - STATICCALL: Read-only (sets is_static, no value)

        5. **Return Data Semantics**:
           - CALL: Clear return_data at START, set from result.output
           - CREATE: Clear return_data at START, set ONLY on failure
           - Write to memory from result.output (up to out_length)

        6. **CREATE Gas**:
           - Base: 32000
           - Init code: 2 gas × words (EIP-3860, Shanghai+)
           - Memory expansion: For reading init code from memory
           - Hash cost (CREATE2): 6 gas × words for keccak256

        7. **SELFDESTRUCT** (EIP-6780, Cancun+):
           - Pre-Cancun: Always delete account
           - Cancun+: ONLY delete if created in same transaction
           - Check evm.created_accounts.contains(frame.address)
           - Always transfer balance (even if not deleting)

        8. **SELFDESTRUCT Gas**:
           - Base: 5000
           - Cold beneficiary: +2600 (Berlin+)
           - New account: +25000 if balance > 0 AND beneficiary doesn't exist
           - Refund: 24000 (pre-London), 0 (London+)

        9. **Precompiles**: Check precompiles.isPrecompile(address, hardfork)
           - Considered to always exist (no new account cost)
           - Handled by evm.inner_call() automatically

        10. **Static Context Violations**:
            - CALL with value > 0: Error in static context
            - CREATE/CREATE2: Error in static context
            - SELFDESTRUCT: Error in static context (AFTER gas charge)

        11. **Account Existence Check**:
            - Use evm.host if available: h.getBalance/getCode/getNonce
            - Fallback: evm.balances/code/nonces
            - Account exists if: balance > 0 OR code.len > 0 OR nonce > 0

        12. **Memory Expansion for Calls**:
            - Calculate max of input_end and output_end
            - Charge expansion ONCE for both regions
            - Update memory_size immediately after charging
      </critical-details>

      <sub-phases>
        <sub-phase id="6.1">
          <name>CREATE Operations</name>
          <ops>CREATE CREATE2</ops>
          Simpler than calls - no memory output, just init code input.
        </sub-phase>

        <sub-phase id="6.2">
          <name>CALL Operations</name>
          <ops>CALL CALLCODE</ops>
          Full call machinery with value transfers.
        </sub-phase>

        <sub-phase id="6.3">
          <name>Delegate Calls</name>
          <ops>DELEGATECALL STATICCALL</ops>
          No value, context preservation.
        </sub-phase>

        <sub-phase id="6.4">
          <name>SELFDESTRUCT</name>
          <ops>SELFDESTRUCT</ops>
          Balance transfer + deletion logic (EIP-6780 awareness).
        </sub-phase>
      </sub-phases>

      <deliverables>
        @guillotine/src/instructions/system.{zig,test.zig}
      </deliverables>

      <success>9 instructions, nested calls work, gas forwarding correct, EIP-6780 handled</success>
    </phase>
    <phase id="7"><name>Bytecode Module</name><status>PLACEHOLDER - STOP AND PLAN FIRST</status><source>@guillotine-mini/src/bytecode.zig</source></phase>
    <phase id="8"><name>Dispatch/Jump (6 ops)</name><status>PLACEHOLDER - STOP AND PLAN FIRST</status><source>@guillotine-mini/src/instructions/handlers_control_flow.zig</source></phase>
    <phase id="9"><name>EIPs/Hardfork</name><status>PLACEHOLDER - STOP AND PLAN FIRST</status><source>@guillotine-mini/src/hardfork.zig</source></phase>
    <phase id="10"><name>Tracing</name><status>PLACEHOLDER - STOP AND PLAN FIRST</status><source>@guillotine-mini/src/trace.zig</source></phase>
    <phase id="11"><name>Frame Module</name><status>PLACEHOLDER - STOP AND PLAN FIRST</status><sources>@guillotine-mini/src/frame.zig, @guillotine/src/frame/frame.zig</sources></phase>
    <phase id="12"><name>Evm Module</name><status>PLACEHOLDER - STOP AND PLAN FIRST</status><sources>@guillotine-mini/src/evm.zig, @guillotine/src/evm.zig</sources></phase>
    <phase id="13"><name>E2E Tests</name><status>PLACEHOLDER - STOP AND PLAN FIRST</status></phase>
  </phases>

  <troubleshooting>
    <when-to-stop>
      ⚠️ STOP IMMEDIATELY if you encounter phase with status="PLACEHOLDER - STOP AND PLAN FIRST" ⚠️

      PLACEHOLDER means:
      - Interface not yet designed
      - Pattern may need adjustment
      - MUST discuss with user before proceeding

      What to do:
      1. Complete current phase completely (all sub-phases + commit)
      2. Report what worked and what's blocked
      3. Ask user for guidance on next phase
      4. DO NOT attempt to continue past PLACEHOLDER phases
    </when-to-stop>

    <common-questions>
      <q>Should I remove this consumeGas() call?</q>
      <a>
        - If argument is GasConstants.SomeConstant → YES, remove
        - If argument is a variable from frame method call → Check if method consumes internally
        - If has hardfork checks → Likely dynamic gas, keep the logic
        - When unsure: Keep it, note in comment "// TODO: verify if this should be removed"
      </a>

      <q>Instruction needs frame.someMethod() that doesn't exist yet</q>
      <a>
        1. Add stub to Frame.zig: pub fn someMethod(self: *Self, args...) ReturnType { @panic("TODO"); }
        2. Document expected behavior in comment
        3. Continue with instruction (tests will skip or expect panic)
        4. Later phases will implement the stub
      </a>

      <q>Source code has getEvm() calls - what do I do?</q>
      <a>
        Keep them! The pattern is:
        - frame.getEvm() delegates to parent EVM state
        - frame doesn't own balances/code/storage - just provides access
        - Add getEvm() stub to Frame if needed: pub fn getEvm(self: *Self) *EvmType { @panic("TODO"); }
      </a>

      <q>Test fails but I copied exactly from source</q>
      <a>
        Check these common mistakes:
        1. Did you change variable names? (a→top breaks stack order)
        2. Did you reorder operations? (Must match source exactly)
        3. Did you remove dynamic gas by accident? (memoryExpansionCost, etc.)
        4. Stack semantics: first pop = TOP, second pop = SECOND (for SUB: second - top)
      </a>

      <q>Should I implement Phase 2 since Phase 1 worked great?</q>
      <a>
        Check the phase status:
        - If status="PLACEHOLDER - STOP AND PLAN FIRST": STOP, don't proceed
        - If phase has detailed <pattern> and <ops>: Ready to implement
        After each phase: Commit, verify tests pass, then proceed to next phase if ready.
      </a>

      <q>How do I access EVM state (balances, storage, etc.)?</q>
      <a>
        Use frame.getEvm() pattern:
        - const evm = frame.getEvm();
        - Access: evm.balances, evm.code, evm.storage, evm.origin, etc.
        - Dynamic gas: const cost = try evm.accessAddress(addr);
        - Frame NEVER owns state - only provides typed access to parent EVM
      </a>

      <q>What's the difference between storage.get() and storage.getOriginal()?</q>
      <a>
        SSTORE refund calculation requires THREE values:
        - original_value: evm.storage.getOriginal(addr, key) - value at transaction START
        - current_value: evm.storage.get(addr, key) - value before this SSTORE
        - new_value: from stack - value being written
        Only original_value uses getOriginal(). Don't confuse them!
      </a>

      <q>Why does SSTORE have different costs in different hardforks?</q>
      <a>
        SSTORE evolved through 4 major EIPs:
        - Pre-Istanbul: Simple (20000 set, 5000 update, 15000 refund)
        - Istanbul (EIP-2200): Net gas metering with dirty tracking
        - Berlin (EIP-2929): Cold/warm access (2100/100 gas)
        - London (EIP-3529): Reduced refunds (4800 instead of 15000)
        Each hardfork stacks on previous. Copy Python logic EXACTLY - don't simplify.
      </a>

      <q>Memory operations failing - what am I missing?</q>
      <a>
        Common mistakes:
        1. Not calling memoryExpansionCost() before access
        2. Not updating memory_size after expansion
        3. MCOPY: Forgetting temporary buffer for overlapping regions
        4. CALLDATACOPY/CODECOPY: Charging expansion for source (only charge for destination)
        5. Word alignment: memory_size must be multiple of 32
      </a>

      <q>How does gas forwarding work for CALL/CREATE?</q>
      <a>
        EIP-150 (Tangerine Whistle) pattern:
        1. Calculate ALL costs FIRST (access + value + new_account + memory)
        2. Subtract from gas_remaining to get "post-charge" state
        3. Calculate forwardable: all but 1/64th of post-charge
        4. THEN charge the cost
        5. For value transfers: add 2300 stipend (FREE, not charged)
        Order matters! Calculate forwarding BEFORE charging.
      </a>
    </common-questions>
  </troubleshooting>

  <success-criteria>
    Phase 1: 74 instructions (stack-only), tests pass, zero static gas/PC in instructions
    Phase 2: 17 instructions (context), getEvm() interface working, hardfork-aware gas
    Phase 3: 5 instructions (memory), reuse Phase 1.1 interface, MCOPY handles overlaps
    Phase 4: 4 instructions (storage), SSTORE matches Python exactly, transient storage works
    Phase 5: 5 instructions (log), shared LOG0-4 implementation, topic order correct
    Phase 6: 9 instructions (system), nested calls work, gas forwarding correct, EIP-6780 handled
    Overall: Single canonical guillotine implementation, all 114 instructions merged
  </success-criteria>
</task>
```

## Update This Prompt

Update if changes needed for new agents.

**ARGUMENTS**: Single context. Start at beginning. TDD. Commit on solid progress (@chop/.claude/commands/commit.md).
