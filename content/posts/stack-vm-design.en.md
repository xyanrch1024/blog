---
title: 'Building a Stack-Based VM: Design Walkthrough'
date: 2026-07-06T14:00:00+08:00
draft: false
cover:
  image: "https://picsum.photos/id/84/800/400"
  alt: "Stack VM"
categories: ["Tech"]
tags: ["Compiler", "VM"]
summary: "A deep dive into the design of a stack-based virtual machine, covering architecture, instruction set, execution model, and memory management."
---

I recently built a stack-based virtual machine in C++ and wrote a detailed design document. This post is a summary of that document.

Source code: https://github.com/xyanrch1024/VM

## Why Stack-Based

Two mainstream VM architectures:

| Feature | Stack | Register |
|---------|-------|----------|
| Instruction size | Short (1-3 bytes) | Longer |
| Compiler backend | Minimal | Needs register allocation |
| Interpretation | Intuitive, fewer bugs | ~30% fewer instructions |
| JIT friendliness | Harder | Natural fit |

I chose **stack-based** because for pure interpretation, it's simpler, easier to code-generate, and less error-prone.

## Architecture

The VM instance contains:

- **Operand stack** — `vector<Value>`, holds locals + temporaries
- **Call stack** — `vector<CallFrame>`, manages function calls
- **String table** — intern pool for O(1) string comparison
- **Function table** — all compiled functions
- **PC / FP** — program counter and frame pointer

## Value System

Each Value is 8 bytes: 1-byte type tag + 7-byte payload:

```
┌──────┬──────────────────────────────┐
│ type │           value               │
│ 1B   │           7B                  │
├──────┼──────────────────────────────┤
│ NIL  │  padding                      │
│ BOOL │  bool                         │
│ INT  │  int64_t                      │
│FLOAT │  double                       │
│STRING│  void* (→ string table)       │
└──────┴──────────────────────────────┘
```

Type promotion: `INT + FLOAT → FLOAT`, string `+` does concatenation.

## Instruction Set (40 instructions)

Each instruction: 1-byte opcode + variable-length operands.

| Category | Count | Examples |
|----------|-------|----------|
| Constants | 5 | `OP_CONSTANT`, `OP_NIL` |
| Stack ops | 5 | `OP_DUP`, `OP_SWAP`, `OP_ROT` |
| Locals | 10 | `OP_LOAD`, `OP_STORE_0~3` |
| Arithmetic | 7 | `OP_ADD`, `OP_SUB`, `OP_MUL` |
| Comparison | 6 | `OP_EQ`, `OP_LT`, `OP_GE` |
| Bitwise | 6 | `OP_BIT_AND`, `OP_SHL` |
| Logic | 1 | `OP_NOT` |
| Control flow | 4 | `OP_JMP`, `OP_JZ`, `OP_LOOP` |
| Function call | 2 | `OP_CALL`, `OP_RET` |
| Other | 4 | `OP_PRINT`, `OP_HALT` |

## Execution Model

The operand stack has two regions:

```
┌───────────────────┐
│   Locals           │  [fp, fp+numLocals)
│   slot 0,1,...,N  │  accessed via LOAD/STORE
├───────────────────┤
│   Expression stack │  [fp+numLocals, sp]
│   Temp values      │  PUSH/POP operations
└───────────────────┘
```

On function call: allocate locals → push CallFrame → execute → RET cleans up.

## Memory Management: String Interning

Every string constant is interned on load:

```
Exists in hash map? → return existing pointer (O(1) compare)
Not found? → create copy, add to table, return stable pointer
```

## Compilation Examples

**1 + 2 × 3 bytecode:**

```
OP_CONSTANT  0   ; push 1
OP_CONSTANT  1   ; push 2
OP_CONSTANT  2   ; push 3
OP_MUL           ; 2 × 3 = 6
OP_ADD           ; 1 + 6 = 7
OP_PRINTLN       ; print 7
OP_HALT
```

**Recursive factorial(5):**

Similar to typical compiler IR — LOAD/STORE for locals, CALL/RET for function calls, LOOP for back-jumps.

## Test Coverage

13 tests covering: arithmetic, locals, conditionals, loops, recursion, floats, strings, comparisons, bitwise, stack ops, and double-recursion (fibonacci).

## Future Optimizations

- **Top-of-stack caching** — cache TOS in a register
- **Computed goto** — replace switch with jump table
- **Inline caching** — inline frequently called functions
- **JIT compilation** — compile hot bytecode to native code

## Conclusion

This design document describes a complete, working stack-based VM. The codebase is compact (C++17, 8 files) but covers the core paradigms of VM design. A good reference for anyone interested in compilers or language implementation.

Full source: https://github.com/xyanrch1024/VM
