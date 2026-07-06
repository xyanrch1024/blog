---
title: '用 2000 行 C++ 写一个类 Lua 语言的前端'
date: 2026-07-06T15:00:00+08:00
draft: false
cover:
  image: "https://picsum.photos/id/160/800/400"
  alt: "Compiler"
categories: ["技术"]
tags: ["编译器", "VM", "Lua"]
summary: "从词法分析、语法分析到字节码编译，完整实现一个类 Lua 语言的前端。"
---

前端是语言实现的第一关：把源码字符串变成可执行的字节码。这篇文章总结我为栈式虚拟机写的 kai 语言前端——一个类 Lua 的脚本语言。

项目代码：https://github.com/xyanrch1024/VM

---

## 整体流程

```
源码 → 词法分析 (Lexer) → Token 流 → 语法分析 (Parser) → AST → 编译 (Compiler) → Bytecode → VM
```

三个模块分工明确：

| 模块 | 输入 | 输出 | 行数 |
|------|------|------|------|
| Lexer | 源文件字符串 | Token 流 | ~200 |
| Parser | Token 流 | AST | ~600 |
| Compiler | AST | Function (字节码) | ~400 |
| 合计 | | | ~1200 |

此外还需要 AST 节点定义 (~200 行) 和内置函数注册 (~100 行)，全前端约 1500 行。

---

## 语言设计：精简版 Lua

kai 是 Lua 的精简方言，保留核心哲学——简约、灵活、嵌入友好。

| 特性 | Lua | kai |
|------|-----|------|
| 返回值 | 多返回值 | 单返回值 |
| 变量 | 默认全局 | 仅 `local` |
| 元表/协程/goto | 支持 | 暂不支持 |
| 泛型 for | 支持 | 仅数值 for |

### EBNF 语法（节选）

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

### 运算符优先级

```
1.  ()  .  []          -- 作用域/索引
2.  #  -  not          -- 一元
3.  ^                  -- 幂 (右结合)
4.  *  /  %            -- 乘法
5.  +  -               -- 加法
6.  ..                 -- 字符串拼接
7.  < > <= >= == ~=    -- 比较
8.  and
9.  or
```

---

## 词法分析 (Lexer)

手动实现的有限状态机，逐个字符扫描。

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

关键字表用 hash 映射，标识符查表决定是关键字还是普通 name。

处理点：`..`（拼接）、`...`（变长参数）、`==` 和 `=` 的区分、注释跳过。

---

## 语法分析 (Parser)

递归下降解析器，每个语法规则对应一个函数。

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

左右递归通过运算符优先级表控制，避免手写层层嵌套。

每个 `parseStat` 函数内嵌对应的语法规则，代码结构与 EBNF 基本一致：

```cpp
// 'if' expr 'then' block { 'elseif' expr 'then' block } [ 'else' block ] 'end'
void Parser::parseIf() {
    std::vector<Expr*> conds;
    std::vector<Stmt*> bodies;
    conds.push_back(parseExpr());
    expect(TK_THEN);
    bodies.push_back(parseBlock());
    while (match(TK_ELSEIF)) { ... }
    if (match(TK_ELSE)) bodies.push_back(parseBlock());
    expect(TK_END);
    new IfStmt(conds, bodies, elseBody);
}
```

---

## AST 定义

使用带 union 的 C++ 结构体，避免虚函数开销。

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

Stmt 同理。这种 tagged union 比继承多态更紧凑，遍历时 switch 分发即可。

---

## 编译器 (AST → Bytecode)

核心是 `compileExpr` 和 `compileStmt` 两个递归函数。

### 表达式编译表

| 源码 | 生成字节码 |
|------|-----------|
| `42` | `OP_CONSTANT <int:42>` |
| `x` | `OP_LOAD <slot>` |
| `a + b` | `a b OP_ADD` |
| `a and b` | `a OP_JZ →end OP_POP b` (短路) |
| `f(x)` | `x <funcIdx> OP_CALL 1` |
| `t[k]` | `t k OP_GET_INDEX` |

### 短路逻辑

`and` 和 `or` 通过跳转实现短路：

```
a and b:
  <a>
  OP_JZ → end       ; a 为假则跳过 b
  OP_POP
  <b>
→ end:

a or b:
  <a>
  OP_JNZ → end      ; a 为真则跳过 b
  OP_POP
  <b>
→ end:
```

### 作用域管理

编译器维护一个 `Scope` 栈：

```cpp
struct Local { const char* name; int depth; int slot; };
vector<Scope> scopes;

void pushScope() { scopes.push_back({scopeDepth++}); }
void popScope() {
    // 发出 POP 清理超出作用域的变量
    for (auto& local : scopes.back().locals)
        emitByte(OP_POP);
    scopes.pop_back();
}
```

变量查找：`resolveLocal(name)` 从内到外遍历作用域链，返回 slot 编号。

### 控制流编译

**if 语句**：条件编译 → `OP_JZ` 跳分支 → 分支体 → `OP_JMP` 跳过剩余分支 → 回填跳转地址。

**while 循环**：记录 loop start → 编译条件 → `OP_JZ` → 编译体 → `OP_LOOP` 回跳 → 回填。

**for 循环**：展开为 while，引入内部变量存储 limit 和 step。

### 函数编译

```
function fact(n)
  if n <= 1 then return 1 end
  return n * fact(n - 1)
end
```

编译过程：
1. 创建新的 `Function` 对象
2. 参数 `n` 分配到 slot 0
3. 编译函数体、递归调用自己
4. 生成的字节码存入独立的 Chunk
5. 外层发出 `OP_CLOSURE <funcIdx>`

### 编译示例完整追踪

```kai
function fact(n)
  if n <= 1 then return 1 end
  return n * fact(n - 1)
end
print(fact(5))
```

生成字节码：

**main 函数：**
```
OP_CLOSURE  0       ; 创建闭包 (fact)
OP_STORE_0          ; fact = closure
OP_POP
OP_CONSTANT  0      ; push 5
OP_CONSTANT  1      ; push funcIdx(0)
OP_CALL      1      ; fact(5)
OP_PRINTLN
OP_HALT
```

**fact 函数：**
```
OP_LOAD_0           ; push n
OP_CONSTANT  0      ; push 1
OP_LE               ; n <= 1?
OP_JZ        → 7    ; 假则跳过 return
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

## VM 扩展

前端需要后端配合新增指令：

| Opcode | 用途 |
|--------|------|
| `OP_GET_INDEX` | `t[k]` 索引读取 |
| `OP_SET_INDEX` | `t[k] = v` 索引写入 |
| `OP_NEW_TABLE` | `{}` 创建表 |
| `OP_CLOSURE` | 创建闭包 |
| `OP_GET_UPVALUE` | 读取 upvalue |
| `OP_SET_UPVALUE` | 写入 upvalue |

Value 类型扩展 `TABLE` 和 `CLOSURE`，Table 实现为 `unordered_map<Value, Value>`。

---

## 分阶段实现

| Phase | 内容 | 行数 | 测试 |
|-------|------|------|------|
| 1 | 词法 + 语法 + 表达式 | ~400 | `print(1 + 2 * 3)` |
| 2 | 控制流 + 变量 | ~300 | if/while/for/短路逻辑 |
| 3 | 函数 + 闭包 | ~200 | 递归 factorial/fib |
| 4 | Table + 内置函数 | ~200 | 表构造器/索引 |

每个阶段都有测试文件 + 预期输出，通过 diff 验证。

---

## 测试策略

```bash
./build/vm tests/arithmetic.kai > /tmp/out
diff /tmp/out tests/arithmetic.expected
```

测试目录：
```
tests/
├── arithmetic.kai     -- 算术运算
├── variables.kai      -- 局部变量 + 作用域
├── if.kai             -- 条件分支
├── while.kai          -- 循环
├── for.kai            -- 数值 for
├── recursion.kai      -- 递归
├── table.kai          -- 表操作
└── ...
```

---

## 总结

这个前端从零构建了一个完整编程语言的第一步：词法分析 → 语法分析 → 字节码编译。

- 词法分析器 ~200 行，手动状态机
- 解析器 ~600 行，递归下降，运算符优先级表
- 编译器 ~400 行，AST → 40 条指令的字节码
- 全前端 ~1500 行 C++17

加上之前实现的栈式虚拟机后端，已经能运行完整的程序了。

完整源码：https://github.com/xyanrch1024/VM
