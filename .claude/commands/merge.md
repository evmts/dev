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
      
      Dispatch-based (vs PC), optimized, synthetic opcodes.
      Goal: shared instructions for BOTH architectures.
    </target>
  </context>

  <implementation-context>
    guillotine-mini: Simple. Handlers do: consumeGas() → stack ops → pc++. @guillotine-mini/src/instructions/handlers_*.zig
    guillotine: Dispatch-based. Handlers: beforeInstruction() → stack ops → tail call. @guillotine/src/instructions/handlers_*.zig
    new-approach: Shared @guillotine/src/instructions/. Generic over FrameType. No gas/PC (caller's job). Pure stack ops.
  </implementation-context>

  <phases>
    <phase id="1">
      <name>Stack-Only Instructions (TDD)</name>
      <goal>~74 opcodes, shared implementations</goal>

      <pattern>
        pub fn OpInstruction(comptime FrameType: type) type {
            return struct {
                pub fn run(frame: *FrameType) FrameType.Error!void {
                    // ⚠️ COPY FROM GUILLOTINE-MINI ⚠️
                    // Remove: consumeGas(), frame.pc += 1
                    // Change: frame.popStack() → frame.stack.pop()
                    // Keep: variable names, order, logic IDENTICAL
                }
            };
        }
        
        Frame interface: frame.stack.{pop,push,peek,set_top,dup_n,swap_n}() -> Error!...
        Errors: {StackOverflow,StackUnderflow,OutOfGas,OutOfBounds,...}
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

    <phase id="2"><name>Context (9 ops)</name><status>PLACEHOLDER</status><source>@guillotine-mini/src/instructions/handlers_context.zig</source></phase>
    <phase id="3"><name>Memory (6 ops)</name><status>PLACEHOLDER</status><source>@guillotine-mini/src/instructions/handlers_memory.zig</source></phase>
    <phase id="4"><name>Storage (4 ops)</name><status>PLACEHOLDER</status><source>@guillotine-mini/src/instructions/handlers_storage.zig</source></phase>
    <phase id="5"><name>Log (5 ops)</name><status>PLACEHOLDER</status><source>@guillotine-mini/src/instructions/handlers_log.zig</source></phase>
    <phase id="6"><name>System (9+ ops)</name><status>PLACEHOLDER</status><source>@guillotine-mini/src/instructions/handlers_system.zig</source></phase>
    <phase id="7"><name>Bytecode Module</name><status>PLACEHOLDER</status><source>@guillotine-mini/src/bytecode.zig</source></phase>
    <phase id="8"><name>Dispatch/Jump (6 ops)</name><status>PLACEHOLDER</status><source>@guillotine-mini/src/instructions/handlers_control_flow.zig</source></phase>
    <phase id="9"><name>EIPs/Hardfork</name><status>PLACEHOLDER</status><source>@guillotine-mini/src/hardfork.zig</source></phase>
    <phase id="10"><name>Tracing</name><status>PLACEHOLDER</status><source>@guillotine-mini/src/trace.zig</source></phase>
    <phase id="11"><name>Frame Module</name><status>PLACEHOLDER</status><sources>@guillotine-mini/src/frame.zig, @guillotine/src/frame/frame.zig</sources></phase>
    <phase id="12"><name>Evm Module</name><status>PLACEHOLDER</status><sources>@guillotine-mini/src/evm.zig, @guillotine/src/evm.zig</sources></phase>
    <phase id="13"><name>E2E Tests</name><status>PLACEHOLDER</status></phase>
  </phases>

  <success-criteria>
    Phase 1: 74 instructions, tests pass, zero gas/PC
    Overall: Single guillotine supporting multiple modes
  </success-criteria>
</task>
```

## Update This Prompt

Update if changes needed for new agents.

**ARGUMENTS**: Single context. Start at beginning. TDD. Commit on solid progress (@chop/.claude/commands/commit.md).
