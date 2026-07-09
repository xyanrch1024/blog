---
title: 'Building a REPL for a Scripting Language'
date: 2026-07-09T16:00:00+08:00
draft: false
categories: ["Tech"]
tags: ["Compiler", "VM", "REPL", "Lua"]
summary: "Designing an interactive Read-Eval-Print Loop for the kai scripting language — multi-line input, expression auto-printing, state persistence, and graceful error recovery."
---

The **REPL** (Read-Eval-Print Loop) is the gateway to any interactive language. It's where beginners experiment, where developers debug snippets, and where the language feels alive. This post walks through the design of the REPL for **kai**, a Lua-like scripting language running on a stack-based VM.

Source code: https://github.com/xyanrch1024/VM

---

## Design Goals

1. **Multi-line input** — functions, if/while/repeat blocks spanning multiple lines
2. **Expression auto-printing** — bare expressions like `1 + 2` automatically print their result
3. **State persistence** — variables declared in one line persist across subsequent lines
4. **Graceful error recovery** — errors don't crash the REPL; the user keeps typing

---

## Architecture: Data Flow

```
stdin → line buffer → isCompleteInput()?
  ├── No  → ">> " prompt → read more
  └── Yes → parseQuiet() test → try print() wrap → runSource(accumulated)
```

The core functions live in `main.cpp`:

| Function | Role |
|----------|------|
| `isCompleteInput(src)` | Bracket + keyword balance heuristic |
| `runSource(src, vm)` | Parse → compile → interpret |
| `Parser::parseQuiet()` | Parse without stderr output (silent test) |

---

## Multi-line Input Detection

### `isCompleteInput()`

This function scans text to determine syntactic completeness by checking three dimensions:

#### Bracket Balance
Character-by-character scan tracking `( )`, `[ ]`, `{ }` depths. Skips inside string literals (`"..."`) and line comments (`--`).

#### Keyword Balance
Converts to lowercase and scans for standalone keywords:

| Opening (increment) | Closing (decrement) |
|---------------------|---------------------|
| `function`, `if`, `while`, `for`, `repeat`, `do` | `end`, `until` |

A keyword is "standalone" only when surrounded by non-alpha characters — so `endless` doesn't falsely close a block.

#### Completeness Condition

```cpp
bool complete = (parenDepth <= 0)
             && (bracketDepth <= 0)
             && (braceDepth <= 0)
             && (blockDepth <= 0);
```

All depths must be ≤ 0. Negative values are allowed (a stray `)` with no `(` won't fool the heuristic — though the real parser will catch it later).

### Prompt Display

```
> 1 + 2           # Primary prompt for first line
>> if x then      # Secondary prompt for continuation lines
>>   print(x)
>> end
```

---

## Expression Auto-printing

### The Problem

In kai, pure expressions like `1 + 2` are valid parse trees but the compiler rejects them as statements with "expression has no effect". In a REPL, users expect to type `1 + 2` and see `3` printed.

### Detection Algorithm

When input is complete, the REPL runs a two-step test:

1. **Try parsing as-is** (with `parseQuiet()`):
   - If it produces valid statements → use the original input
   - Handles: `local x = 10`, `print("hi")`, `x = 5`, `if ... end`

2. **If step 1 fails, try wrapping in `print(...)`**:
   - If `print(original)` parses cleanly → use the wrapped version
   - Handles: `1+2`, `"hello"`, `x`, `f(3)` — anything that's an expression

3. **Fallback**: if neither works, use the original input (the real error message will show).

### Quiet Parsing

To avoid double-printing confusing errors during the test parse, we redirect stderr to `/dev/null`:

```cpp
std::vector<Stmt*> Parser::parseQuiet() {
    FILE* oldStderr = stderr;
    FILE* devnull = fopen("/dev/null", "w");
    if (devnull) stderr = devnull;
    auto result = parse();
    if (devnull) {
        stderr = oldStderr;
        fclose(devnull);
    }
    return result;
}
```

Error messages are only shown to the user during the real compilation in `runSource()`.

---

## State Persistence Strategy

The REPL accumulates all entered source in a `std::string accumulated`. Each time the user submits a complete input, the *entire* accumulated source is recompiled and re-executed from scratch.

```
Line 1:  local x = 10    → accumulated = "local x = 10"
Line 2:  print(x)        → accumulated = "local x = 10\nprint(x)"
                          → recompile & run everything
```

This approach mirrors early BASIC REPLs and has a clear trade-off:

| Pro | Con |
|-----|-----|
| Simple implementation | All side effects re-executed on every new line |
| No compiler changes needed | `print()` output repeats as lines grow |
| Locals naturally persist | Performance degrades with session length |

For a learning/debugging environment, this simplicity is worth the cost. A future improvement could adopt incremental compilation with a persistent symbol table.

---

## REPL Loop Pseudocode

```
accumulated = ""       // persistent source string
buffer = ""            // current input being built
inMultiLine = false

loop:
  prompt = inMultiLine ? ">> " : "> "
  print(prompt)
  line = readLine(stdin)
  if EOF:
    if inMultiLine && buffer not empty: runSource(buffer, vm)
    break
  if not inMultiLine && line in {"exit", "quit"}: break

  buffer += "\n" + line
  if buffer is all whitespace: continue

  if isCompleteInput(buffer):
    toRun = ""

    // Try as-is
    parser = Parser(buffer)
    stmts = parser.parseQuiet()
    if stmts not empty and no error:
      toRun = buffer

    // Try wrapped in print()
    if toRun empty:
      parser = Parser("print(" + buffer + ")")
      stmts = parser.parseQuiet()
      if stmts not empty and no error:
        toRun = "print(" + buffer + ")"

    // Fallback
    if toRun empty:
      toRun = buffer

    accumulated += "\n" + toRun
    runSource(accumulated, vm)

    buffer = ""
    inMultiLine = false
  else:
    inMultiLine = true
```

---

## Limitations & Future Work

### Current Limitations

| Limitation | Cause | Impact |
|---|---|---|
| **Side-effect re-execution** | State persistence recompiles all input each time | `print()` output repeated on every new line |
| **Heuristic completeness check** | Text-based, not grammar-based | False positives/negatives possible with unusual syntax |
| **No history** | No readline/libedit integration | No up-arrow recall |
| **No tab completion** | No symbol table exposed to REPL | Manual typing only |

### Potential Improvements

**1. Incremental compilation with persistent scope**
Keep the Compiler's symbol table alive across lines. Each line compiles into a block appended to the same function. Requires significant changes to `CompileState` lifecycle.

**2. Readline support**
Link against `libreadline` or `libedit` for line editing, history, and tab completion. Replace `std::getline()` with `readline()`.

**3. Grammar-aware completeness check**
Instead of the heuristic `isCompleteInput()`, use the actual parser with a special "expect more" error mode. If the parser fails with `unexpected EOF`, prompt for more input. Would require modifications to `Lexer::next()` and `Parser::consume()`.

**4. Persistent REPL namespace**
Replace the accumulated-source approach with a real symbol table that persists in the VM. Each line becomes a separate function sharing the same top-level local scope, using a hidden table to store REPL-declared variables.

---

## Summary

The kai REPL is a pragmatic, minimal design that balances implementation effort against user experience:

- **Multi-line input** via a heuristic balance checker
- **Expression auto-printing** via silent parse-and-wrap
- **State persistence** via full-source recompilation (simple but wasteful)
- **Error recovery** naturally handled by the loop structure

It's not a production-grade REPL — no readline, no tab completion, no incremental compilation — but it gets the job done for language experimentation. The full codebase, including the lexer, parser, compiler, and VM, is open source.

Full source: https://github.com/xyanrch1024/VM
