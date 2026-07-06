---
title: 'Building a Lua-like Language Frontend in C++'
date: 2026-07-06T15:00:00+08:00
draft: false
cover:
  image: "https://picsum.photos/id/160/800/400"
  alt: "Compiler"
categories: ["Tech"]
tags: ["Compiler", "VM", "Lua"]
summary: "From lexer and parser to bytecode compiler — building a Lua-like language frontend for a stack-based VM."
---

The frontend is the first pass of any language implementation: turning source code into executable bytecode. This post summarizes the frontend of **kai**, a Lua-like scripting language for my stack-based VM.

Source code: https://github.com/xyanrch1024/VM

---

## Pipeline

```
Source → Lexer → Token stream → Parser → AST → Compiler → Bytecode → VM
```

| Module | Input | Output | Lines |
|--------|-------|--------|-------|
| Lexer | source string | Token stream | ~200 |
| Parser | Token stream | AST | ~600 |
| Compiler | AST | Function (bytecode) | ~400 |
| Total | | | ~1200 |

Plus AST definitions (~200 lines) and built-in functions (~100 lines) brings the full frontend to ~1500 lines of C++17.

---

## Language Design: Lua Simplified

kai is a streamlined Lua dialect that keeps the core philosophy — simplicity, flexibility, embeddability.

| Feature | Lua | kai |
|---------|-----|------|
| Returns | Multiple | Single |
| Variables | Global by default | `local` only |
| Metatable/Coroutine/goto | Supported | Not yet |
| Generic for | Supported | Numeric for only |

### EBNF Grammar (excerpt)

```
program   = { stat }
stat      = 'local' name '=' expr
          | name '=' expr
          | 'if' expr 'then' block 'end'
          | 'while' expr 'do' block 'end'
          | 'function' name '(' [ namelist ] ')' block 'end'
          | 'return' [ expr ]
          | functioncall
expr      = nil | true | false | NUMBER | STRING
          | functiondef | tableconstructor
          | prefixexpr | expr binop expr | unop expr
```

### Operator Precedence

```
1.  ()  .  []          -- scope/index
2.  #  -  not          -- unary
3.  ^                  -- exponent (right-assoc)
4.  *  /  %            -- multiplicative
5.  +  -               -- additive
6.  ..                 -- concatenation
7.  < > <= >= == ~=    -- comparison
8.  and
9.  or
```

---

## Lexer

Hand-written finite state machine, scanning character by character.

```cpp
Token Lexer::nextToken() {
    skipWhitespace();
    switch (*cur) {
        case '+': cur++; return {TK_PLUS, line};
        case '-':
            if (peek() == '-') { skipComment(); return nextToken(); }
            cur++; return {TK_MINUS, line};
        case '"': return readString();
        default:
            if (isdigit(*cur)) return readNumber();
            if (isalpha(*cur) || *cur == '_') return readNameOrKeyword();
            error("unexpected symbol '%c'", *cur);
    }
}
```

Keywords are resolved via a hash map lookup after reading an identifier.

Edge cases handled: `..` (concat), `...` (varargs), `==` vs `=`, comment skipping.

---

## Parser

Recursive descent parser — every grammar rule maps to a function.

```cpp
Expr* Parser::parseExpr(int precedence) {
    Expr* expr = parsePrefix();
    while (precedence < getPrecedence(current().type)) {
        TokenType op = current().type;
        advance();
        Expr* rhs = parseExpr(getRightPrecedence(op));
        expr = new BinaryExpr(expr, op, rhs);
    }
    return expr;
}
```

Left/right recursion is handled by a precedence table, avoiding deeply nested function calls.

---

## AST Definition

Tagged unions in C++ — no virtual dispatch overhead.

```cpp
struct Expr {
    ExprType type;
    int line;
    union {
        double numVal;                              // NUMBER
        const char* strVal;                         // STRING / NAME
        struct { TokenType op; Expr* rhs; } unary;  // UNARY
        struct { TokenType op; Expr* l, *r; } bin;  // BINARY
        struct { Expr* callee; vector<Expr*> args; } call;  // CALL
        struct { Expr* obj; Expr* key; } index;     // INDEX
        struct { vector<const char*> p; Stmt* b; } func;    // FUNCDEF
        struct { vector<TableField> f; } table;     // TABLE
    };
};
```

Same pattern for Stmt. This tagged union approach is more compact than polymorphic classes and dispatches with a simple switch.

---

## Compiler (AST → Bytecode)

Two recursive functions: `compileExpr` and `compileStmt`.

### Expression Compilation Table

| Source | Bytecode |
|--------|----------|
| `42` | `OP_CONSTANT <int:42>` |
| `x` | `OP_LOAD <slot>` |
| `a + b` | `a b OP_ADD` |
| `a and b` | `a OP_JZ →end OP_POP b` (short-circuit) |
| `f(x)` | `x <funcIdx> OP_CALL 1` |
| `t[k]` | `t k OP_GET_INDEX` |

### Short-Circuit Logic

`and` and `or` compile with jump-based short-circuit evaluation:

```
a and b:
  <a>
  OP_JZ → end       ; if a is false, skip b
  OP_POP
  <b>
→ end:

a or b:
  <a>
  OP_JNZ → end      ; if a is true, skip b
  OP_POP
  <b>
→ end:
```

### Scope Management

The compiler maintains a scope stack:

```cpp
struct Local { const char* name; int depth; int slot; };
vector<Scope> scopes;

void pushScope() { scopes.push_back({scopeDepth++}); }
void popScope() {
    for (auto& local : scopes.back().locals)
        emitByte(OP_POP);   // clean up locals
    scopes.pop_back();
}
```

Variable lookup (`resolveLocal(name)`) traverses the scope chain from innermost outward, returning the slot number.

### Control Flow Compilation

**if statement**: condition → `OP_JZ` → branch body → `OP_JMP` → patch jumps.

**while loop**: record loop start → condition → `OP_JZ` → body → `OP_LOOP` → patch.

**for loop**: desugared to while with internal limit/step variables.

### Function Compilation

```
function fact(n)
  if n <= 1 then return 1 end
  return n * fact(n - 1)
end
```

Compilation process:
1. Create new `Function` object
2. Parameter `n` → slot 0
3. Compile body with recursive self-call
4. Bytecode stored in independent Chunk
5. Parent emits `OP_CLOSURE <funcIdx>`

### Bytecode Trace

```kai
function fact(n)
  if n <= 1 then return 1 end
  return n * fact(n - 1)
end
print(fact(5))
```

**main function:**
```
OP_CLOSURE  0       ; create closure (fact)
OP_STORE_0          ; fact = closure
OP_POP
OP_CONSTANT  0      ; push 5
OP_CONSTANT  1      ; push funcIdx(0)
OP_CALL      1      ; fact(5)
OP_PRINTLN
OP_HALT
```

**fact function:**
```
OP_LOAD_0           ; push n
OP_CONSTANT  0      ; push 1
OP_LE               ; n <= 1?
OP_JZ        → 7    ; skip return if false
OP_CONSTANT  1      ; push 1
OP_RET
OP_LOAD_0           ; push n
OP_LOAD_0           ; push n
OP_CONSTANT  2      ; push 1
OP_SUB              ; n - 1
OP_CONSTANT  3      ; push funcIdx(1)
OP_CALL      1      ; fact(n-1)
OP_MUL              ; n * fact(n-1)
OP_RET
```

---

## VM Extensions

The frontend required new instructions in the backend:

| Opcode | Purpose |
|--------|---------|
| `OP_GET_INDEX` | `t[k]` indexed read |
| `OP_SET_INDEX` | `t[k] = v` indexed write |
| `OP_NEW_TABLE` | `{}` create table |
| `OP_CLOSURE` | Create closure |
| `OP_GET_UPVALUE` | Read upvalue |
| `OP_SET_UPVALUE` | Write upvalue |

Value types extended with `TABLE` and `CLOSURE`. Table implemented as `unordered_map<Value, Value>`.

---

## Phased Implementation

| Phase | Content | Lines | Test |
|-------|---------|-------|------|
| 1 | Lex + Parse + Expressions | ~400 | `print(1 + 2 * 3)` |
| 2 | Control flow + Variables | ~300 | if/while/for/short-circuit |
| 3 | Functions + Closures | ~200 | recursive factorial/fib |
| 4 | Table + Builtins | ~200 | table constructor/index |

Each phase has test files + expected output, verified by diff.

---

## Testing

```bash
./build/vm tests/arithmetic.kai > /tmp/out
diff /tmp/out tests/arithmetic.expected
```

Test directory:
```
tests/
├── arithmetic.kai
├── variables.kai
├── if.kai
├── while.kai
├── for.kai
├── recursion.kai
├── table.kai
└── ...
```

---

## Summary

This frontend covers the first steps of building a complete programming language:

- Lexer: ~200 lines, hand-written state machine
- Parser: ~600 lines, recursive descent with precedence table
- Compiler: ~400 lines, AST → 40 instructions of bytecode
- Full frontend: ~1500 lines of C++17

Combined with the stack-based VM backend from the previous post, it can now run complete programs end-to-end.

Full source: https://github.com/xyanrch1024/VM
