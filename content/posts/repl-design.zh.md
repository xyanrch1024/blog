---
title: '为脚本语言构建 REPL 交互环境'
date: 2026-07-09T16:00:00+08:00
draft: false
categories: ["技术"]
tags: ["编译器", "虚拟机", "REPL", "Lua"]
summary: "为 kai 脚本语言设计交互式 Read-Eval-Print Loop——多行输入、表达式自动打印、状态持久化和优雅的错误恢复。"
---

**REPL**（Read-Eval-Print Loop）是任何交互式语言的门户。初学者用它来实验，开发者用它来调试代码片段，它让语言有了生命力。本文介绍 **kai**（一种运行在栈式虚拟机上的 Lua 风格脚本语言）的 REPL 设计。

源代码：https://github.com/xyanrch1024/VM

---

## 设计目标

1. **多行输入**——函数、if/while/repeat 块可以跨越多行
2. **表达式自动打印**——像 `1 + 2` 这样的裸表达式会自动输出结果
3. **状态持久化**——在一行中声明的变量在后续行中可用
4. **优雅的错误恢复**——错误不会让 REPL 崩溃，用户可以继续输入

---

## 架构：数据流

```
stdin → 行缓冲区 → isCompleteInput()?
  ├── 否 → ">> " 提示符 → 继续读取
  └── 是 → parseQuiet() 测试 → 尝试 print() 包装 → runSource(accumulated)
```

核心函数位于 `main.cpp`：

| 函数 | 作用 |
|------|------|
| `isCompleteInput(src)` | 括号 + 关键字平衡检测 |
| `runSource(src, vm)` | 解析 → 编译 → 解释执行 |
| `Parser::parseQuiet()` | 静默解析（不输出 stderr） |

---

## 多行输入检测

### `isCompleteInput()`

该函数通过扫描文本来判断语法完整性，检查三个维度：

#### 括号平衡
逐字符扫描追踪 `( )`、`[ ]`、`{ }` 的深度，跳过字符串字面量（`"..."`）和行注释（`--`）。

#### 关键字平衡
转换为小写后扫描独立关键字：

| 开启关键字（深度+1） | 关闭关键字（深度-1） |
|---------------------|---------------------|
| `function`, `if`, `while`, `for`, `repeat`, `do` | `end`, `until` |

关键字只有在前后都是非字母字符时才被认为是"独立"的——这样 `endless` 就不会错误地关闭一个块。

#### 完整性条件

```cpp
bool complete = (parenDepth <= 0)
             && (bracketDepth <= 0)
             && (braceDepth <= 0)
             && (blockDepth <= 0);
```

所有深度必须 ≤ 0。允许出现负值（例如孤立的 `)` 没有匹配的 `(` 不会欺骗启发式检测——虽然真正的解析器会在后续捕获它）。

### 提示符显示

```
> 1 + 2           # 第一行的主提示符
>> if x then      # 续行的副提示符
>>   print(x)
>> end
```

---

## 表达式自动打印

### 问题

在 kai 中，像 `1 + 2` 这样的纯表达式是有效的解析树，但编译器会以"表达式没有效果"为由拒绝它们作为语句。在 REPL 中，用户输入 `1 + 2` 期望看到 `3` 被打印出来。

### 检测算法

当输入完整时，REPL 执行两步测试：

1. **直接尝试解析**（使用 `parseQuiet()`）：
   - 如果产生有效语句 → 使用原始输入
   - 处理：`local x = 10`、`print("hi")`、`x = 5`、`if ... end`

2. **如果第 1 步失败，尝试包裹在 `print(...)` 中**：
   - 如果 `print(original)` 能解析成功 → 使用包裹版本
   - 处理：`1+2`、`"hello"`、`x`、`f(3)`——任何表达式

3. **回退**：如果都不行，使用原始输入（真实的错误消息会显示给用户）。

### 静默解析

为了避免在测试解析时打印令人困惑的错误消息，我们将 stderr 重定向到 `/dev/null`：

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

错误消息只在 `runSource()` 中的真正编译期间显示给用户。

---

## 状态持久化策略

REPL 将所有输入的源代码累加在一个 `std::string accumulated` 中。每次用户提交完整输入时，整个累加的源代码都会被重新编译并从头执行。

```
第 1 行：local x = 10    → accumulated = "local x = 10"
第 2 行：print(x)        → accumulated = "local x = 10\nprint(x)"
                          → 重新编译并运行全部
```

这种方法类似于早期 BASIC 的 REPL，其优缺点如下：

| 优点 | 缺点 |
|-----|------|
| 实现简单 | 每次新行都重新执行所有副作用 |
| 无需修改编译器 | `print()` 输出会随着行数增长而重复 |
| 局部变量自然保持 | 性能随会话时间增长而下降 |

对于学习/调试环境，这种简单性值得付出代价。未来的改进可以采用增量编译和持久的符号表。

---

## REPL 循环伪代码

```
accumulated = ""       // 持久化的源代码字符串
buffer = ""            // 当前正在构建的输入
inMultiLine = false

循环：
  prompt = inMultiLine ? ">> " : "> "
  打印 prompt
  line = readLine(stdin)
  如果是 EOF：
    如果 inMultiLine 且 buffer 不为空：runSource(buffer, vm)
    退出
  如果不是多行且 line 是 "exit" 或 "quit"：退出

  buffer += "\n" + line
  如果 buffer 全是空白：继续

  如果 isCompleteInput(buffer)：
    toRun = ""

    // 尝试原样解析
    parser = Parser(buffer)
    stmts = parser.parseQuiet()
    如果 stmts 不为空且无错误：
      toRun = buffer

    // 尝试包裹在 print() 中
    如果 toRun 为空：
      parser = Parser("print(" + buffer + ")")
      stmts = parser.parseQuiet()
      如果 stmts 不为空且无错误：
        toRun = "print(" + buffer + ")"

    // 回退
    如果 toRun 为空：
      toRun = buffer

    accumulated += "\n" + toRun
    runSource(accumulated, vm)

    buffer = ""
    inMultiLine = false
  否则：
    inMultiLine = true
```

---

## 局限性及未来工作

### 当前局限性

| 局限 | 原因 | 影响 |
|------|------|------|
| **副作用重执行** | 状态持久化每次重新编译所有输入 | `print()` 输出每次新行都重复 |
| **启发式完整性检查** | 基于文本，不基于语法 | 特殊语法可能误判 |
| **无历史记录** | 未集成 readline/libedit | 无法用上箭头回忆之前的行 |
| **无 Tab 补全** | 未向 REPL 暴露符号表 | 只能手动输入 |

### 可能的改进

**1. 增量编译 + 持久作用域**
保持编译器的符号表跨行存活。每行编译为一个独立块，追加到同一函数中。需要大幅修改 `CompileState` 的生命周期。

**2. Readline 支持**
链接 `libreadline` 或 `libedit`，获得行编辑、历史和 Tab 补全功能。用 `readline()` 替换 `std::getline()`。

**3. 语法感知的完整性检查**
用真正的解析器替代启发式的 `isCompleteInput()`，使用特殊的"期待更多"错误模式。如果解析器因 `unexpected EOF` 失败，提示用户继续输入。需要修改 `Lexer::next()` 和 `Parser::consume()`。

**4. 持久化 REPL 命名空间**
用 VM 中持久化的真实符号表替代累加源代码的方式。每行成为单独的函数，共享相同的顶层局部作用域，使用隐藏表存储 REPL 声明的变量。

---

## 总结

kai REPL 是一个实用主义的最小设计，在实现成本与用户体验之间取得了平衡：

- **多行输入**——通过启发式的括号/关键字平衡检查器
- **表达式自动打印**——通过静默解析和 print 包装
- **状态持久化**——通过全源重新编译（简单但低效）
- **错误恢复**——由循环结构自然处理

这不是一个生产级别的 REPL——没有 readline、没有 Tab 补全、没有增量编译——但它足以完成语言实验的工作。完整代码库（包括词法分析器、解析器、编译器和虚拟机）是开源的。

完整源码：https://github.com/xyanrch1024/VM
