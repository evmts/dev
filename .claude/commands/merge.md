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

    <phase id="2"><name>Context (9 ops)</name><status>PLACEHOLDER - STOP AND PLAN FIRST</status><source>@guillotine-mini/src/instructions/handlers_context.zig</source></phase>
    <phase id="3"><name>Memory (6 ops)</name><status>PLACEHOLDER - STOP AND PLAN FIRST</status><source>@guillotine-mini/src/instructions/handlers_memory.zig</source></phase>
    <phase id="4"><name>Storage (4 ops)</name><status>PLACEHOLDER - STOP AND PLAN FIRST</status><source>@guillotine-mini/src/instructions/handlers_storage.zig</source></phase>
    <phase id="5"><name>Log (5 ops)</name><status>PLACEHOLDER - STOP AND PLAN FIRST</status><source>@guillotine-mini/src/instructions/handlers_log.zig</source></phase>
    <phase id="6"><name>System (9+ ops)</name><status>PLACEHOLDER - STOP AND PLAN FIRST</status><source>@guillotine-mini/src/instructions/handlers_system.zig</source></phase>
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
        NO! Phase 2+ have PLACEHOLDER status meaning STOP AND PLAN FIRST.
        After Phase 1:
        1. Commit your work
        2. Report success
        3. Ask user how to proceed with Phase 2
        DO NOT continue without explicit user guidance.
      </a>
    </common-questions>
  </troubleshooting>

  <success-criteria>
    Phase 1: 74 instructions, tests pass, zero static gas/PC in instructions
    Overall: Single canonical guillotine implementation
  </success-criteria>
</task>
```

## Update This Prompt

Update if changes needed for new agents.

**ARGUMENTS**: Single context. Start at beginning. TDD. Commit on solid progress (@chop/.claude/commands/commit.md).
