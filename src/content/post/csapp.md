---
title: CSAPP 3nd - 汇编与控制
description: "这篇文章介绍了CSAPP第三章的汇编与控制内容，包括寄存器类型、条件码、条件分支、循环、switch语句等。"
publishDate: "7 Apr 2025"
updatedDate: "7 Apr 2025"
tags: ["CSAPP", "assembly", "control flow", "computer science"]
---

## 汇编

### 寄存器类型

- 64位为`r`
- 32位为`e`

| 汇编术语 | 位数 | 字节数 | 示例寄存器             |
| -------- | ---- | ------ | ---------------------- |
| byte     | 8    | 1      | `%al`, `%bl`, `%r8b`   |
| word     | 16   | 2      | `%ax`, `%bx`, `%r8w`   |
| long     | 32   | 4      | `%eax`, `%ebx`, `%r8d` |
| quad     | 64   | 8      | `%rax`, `%rbx`, `%r8`  |

### Control

#### Condition codes

- 条件码
- `cmpq`
- `setX`
  基于条件码的值将单个寄存器的单个字节设置为0或1（取对应条件码低位的数字作为目标寄存器的低位，然后将剩余7位置为0）
- `movzbl`
  x86-64机器上，结果为32位的数字会自动将高位的32位扩展为0，但是对于低于4字节的结果就不会进行扩展，所以会有`movzbl`指令来对单字节（两个字节为`movzwl`）的数字进行扩展到四个字节（z代表zero，b代表byte，l代表long）

#### Conditional branches

**`jX`指令**

| 指令 | 条件描述             | 对应 setX | 标志条件          | 用于比较类型    |
| ---- | -------------------- | --------- | ----------------- | --------------- |
| je   | 相等（equal）        | sete      | ZF = 1            | 有符号 / 无符号 |
| jne  | 不相等（not equal）  | setne     | ZF = 0            | 有符号 / 无符号 |
| jg   | 大于（signed）       | setg      | ZF = 0 且 SF = OF | 有符号          |
| jge  | 大于等于（signed）   | setge     | SF = OF           | 有符号          |
| jl   | 小于（signed）       | setl      | SF ≠ OF           | 有符号          |
| jle  | 小于等于（signed）   | setle     | ZF = 1 或 SF ≠ OF | 有符号          |
| ja   | 高于（unsigned）     | seta      | CF = 0 且 ZF = 0  | 无符号          |
| jae  | 高于等于（unsigned） | setae     | CF = 0            | 无符号          |
| jb   | 低于（unsigned）     | setb      | CF = 1            | 无符号          |
| jbe  | 低于等于（unsigned） | setbe     | CF = 1 或 ZF = 1  | 无符号          |
| js   | 负数（负号）         | sets      | SF = 1            | 通用            |
| jns  | 非负                 | setns     | SF = 0            | 通用            |
| jo   | 溢出（overflow）     | seto      | OF = 1            | 通用            |
| jno  | 无溢出               | setno     | OF = 0            | 通用            |

生成跳转汇编语言的gcc指令(这个指令的意思是编译成汇编语言，-Og表示优化级别为0，-S表示只编译成汇编语言，不链接，-fno-if-conversion表示不使用条件移动指令)。
```bash
gcc -Og -S -fno-if-conversion control.c
```
#### 📘 函数：计算两个 long 整数的差的绝对值

#### C 语言代码：

```c
long absdiff(long x, long y) {
    long result;
    if (x > y)
        result = x - y;
    else
        result = y - x;
    return result;
}
```
汇编代码（AT&T 语法）：
```asm
absdiff:
    cmpq    %rsi, %rdi        # 比较 x : y （x > y?）
    jle     .L4               # 如果 x <= y，跳到 .L4
    movq    %rdi, %rax        # x > y: 把 y 放入 rax
    subq    %rsi, %rax        # rax = y - x
    ret

.L4:                          # x <= y 分支
    movq    %rsi, %rax        # 把 x 放入 rax
    subq    %rdi, %rax        # rax = x - y
    ret
```
如果使用了条件移动指令，那么上面的的代码就会变成：
```asm
absdiff:
    movq   %rdi, %rax        # 把 x 放入 rax
    subq   %rsi, %rax        # rax = x - y 
    movq   %rsi, %rdx        # 把 y 放入 rdx
    subq   %rdi, %rdx        # rdx = y - x
    cmopq  %rsi, %rdi        # 比较 x : y （x > y?）
    cmovle %rdx, %rax        # 如果 x <= y，rax = x - y
```
条件移动会提前计算出所有的结果，然后根据条件码来决定使用哪个结果。
- 对于简单地分支，使用条件移动指令会更快，但是对于复杂的分支，使用条件移动指令会更慢。
- 对于空指针这种情况，不能使用条件移动，会导致程序异常。
  ```
    val = p ? *p:0; // 这里的条件移动会导致空指针异常
  ```
- 对于前一个分支带有副作用的情况，也不能使用条件指令
  ```
    var = x ? x+=3:x*=7; 
  ```
#### Loops
在汇编层级，do-while循环是另外两种循环的基础结构，while和for循环都可以用do-while循环来实现。
```asm
do_while:
    .L2:
        # 循环体
        # ...
        # 判断条件
        cmpq    $0, %rdi       # 比较条件
        jne     .L2            # 如果条件不为0，跳到 .L2
```
```asm
while:
    .L3:
        # 判断条件
        cmpq    $0, %rdi       # 比较条件
        je      .L4            # 如果条件为0，跳到 .L4
        # 循环体
        # ...
        jmp     .L3            # 跳到 .L3
.L4:
```
```asm
for:
    # 初始化
    movq    $0, %rdi         # 初始化条件
    .L5:
        # 判断条件
        cmpq    $10, %rdi      # 比较条件
        jge     .L6            # 如果条件大于等于10，跳到 .L6
        # 循环体
        # ...
        incq    %rdi           # 条件自增1
        jmp     .L5            # 跳到 .L5
.L6:
```
#### Switch Statements
#### ✅ switch 中 case 常量的规则总结：

| 要求                 | 说明                                       |
|----------------------|--------------------------------------------|
| 必须是常量表达式     | 编译时能确定的整数，如 `1`, `3+4`, `#define A 5` |
| 不能重复             | 每个 case 值必须唯一                        |
| 不要求递增或连续     | case 可以无序、稀疏，顺序无关                |
| 类型必须是整型       | 支持 int, char, short, enum 等，不支持 float、指针 |
### 🔁 switch 的跳转表原理

- 跳转表（jump table）是一种将 case 常量值映射为 **代码地址** 的数组。
- 编译器使用 `jmp *table(x)` 实现 O(1) 多分支跳转。
- 只有当 case 值 **连续且数量较多** 时才会启用。

#### 汇编结构概览：

```asm
cmp $max, %x
ja .Ldefault
jmp *.Ltable(,%x,8)

.Ltable:
    .quad .Lcase0
    .quad .Lcase1
    ...
```
### 📘 指令含义（switch 跳转表相关）

| 指令 | 作用 | 含义解释 |
|------|------|----------|
| `ja label` | 条件跳转（无符号） | 如果前面 `cmp` 中无符号大于成立，跳转到 label |
| `jmp *%rdi` | 间接跳转 | 跳转到 `%rdi` 寄存器中存放的地址执行 |
| `jmp *.Ljump_table(,%rdi,8)` | 间接跳转 | 跳转到跳转表中第 `%rdi` 项所存放的地址执行 |

## 🧠 GDB `x` 命令（examine memory）用法速查表

GDB 的 `x` 命令用于查看任意内存地址的内容，支持以多种格式和单位显示，是调试汇编、指针、栈内容的重要工具。

### 🧾 基本语法

x/[个数] [格式] [单位] 地址

- `个数`：显示几个单元（默认1）
- `格式`：显示格式（hex、dec、string等）
- `单位`：每个单元的大小（byte、word等）

---

### 📌 常用格式符号

| 格式符 | 含义               | 示例        |
| ------ | ------------------ | ----------- |
| `x`    | 十六进制（hex）    | `x/4x $rsp` |
| `d`    | 十进制（signed）   | `x/4d addr` |
| `u`    | 十进制（unsigned） | `x/4u addr` |
| `t`    | 二进制             | `x/4t addr` |
| `f`    | 浮点数             | `x/4f addr` |
| `s`    | 字符串（C 风格）   | `x/s $rsi`  |
| `i`    | 汇编指令（反汇编） | `x/i $rip`  |

---

### 📌 常用单位符号

| 单位符 | 单元大小 | 说明                |
| ------ | -------- | ------------------- |
| `b`    | 1 字节   | byte                |
| `h`    | 2 字节   | halfword（少用）    |
| `w`    | 4 字节   | word（32 位）       |
| `g`    | 8 字节   | giant word（64 位） |

---

### 🔍 常用命令示例

| 命令           | 含义                                          |
| -------------- | --------------------------------------------- |
| `x/x 0x601040` | 查看地址 `0x601040` 的 1 个 word（4B）        |
| `x/4x $rsp`    | 查看 `$rsp` 开始的 4 个 word（共16字节）      |
| `x/gx $rbp`    | 查看 `$rbp` 指向的 1 个 giant word（64 位）   |
| `x/8gx $rsp`   | 连续查看 8 个 64 位值（常用于查看栈）         |
| `x/4xb $rax`   | 查看 `$rax` 指向地址的 4 个字节（逐字节显示） |
| `x/s 0x6037d0` | 查看地址处的 C 字符串                         |
| `x/i $rip`     | 查看当前指令（反汇编）                        |

---

### 🎯 实用技巧

- 使用 `$` 可引用寄存器名：如 `$rsp`、`$rbp`、`$rax`
- 小端序：`x86_64` 架构下内存按小端序存储
- 可结合 `/b` 查看字节级别数据结构（如结构体、栈帧等）

---

### 🧰 推荐组合操作

```gdb
x/gx $rsp         # 栈顶64位
x/8x $rsp         # 栈上连续4字（16B）
x/4xb $rax        # 查看寄存器指向地址的4字节（字节级）
x/s $rsi          # 打印字符串指针的内容
x/i $rip          # 查看当前执行指令
```