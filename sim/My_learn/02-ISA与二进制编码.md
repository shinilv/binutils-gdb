# 02 ISA 与二进制编码

## 1. 架构资源

Synacor CPU 有三类存储：

1. 32768 个 word 的内存；
2. 8 个寄存器；
3. 保存 16 位值的逻辑栈。

有效计算值只有 15 位：`0..32767`，即 `0x0000..0x7fff`。算术结果按模 32768 处理。

```text
32758 + 15 = 32773
32773 mod 32768 = 5
```

每个程序 word 在文件中按 16 位小端序保存。例如：

```text
数值 9      -> 09 00
数值 32768  -> 00 80
数值 32769  -> 01 80
```

## 2. 一个 16 位 word 的三种含义

| 编码范围 | 含义 |
|---|---|
| `0..32767` | 字面量 |
| `32768..32775` | R0..R7 |
| `32776..65535` | 非法编码 |

寄存器编码关系：

```text
0x8000 = 32768 = R0
0x8001 = 32769 = R1
...
0x8007 = 32775 = R7
```

必须区分“原始编码”和“解释后的值”。例如 R1 当前保存 65：

```text
原始 word 32769
作为源操作数解释后得到 65
作为目标操作数解释后得到寄存器编号 1
```

这对应源码中的两个函数：

- `interp_num(cpu, raw)`：读取源操作数，返回立即数或寄存器内容；
- `register_num(cpu, raw)`：验证目标必须是寄存器，返回编号 0～7。

## 3. 指令表

| opcode | 格式 | 语义 | word 数/byte 数 |
|---:|---|---|---:|
| 0 | `halt` | 正常停止 | 1 / 2 |
| 1 | `set a b` | `a = b` | 3 / 6 |
| 2 | `push a` | 压栈 | 2 / 4 |
| 3 | `pop a` | 弹栈到 a | 2 / 4 |
| 4 | `eq a b c` | `a = (b == c)` | 4 / 8 |
| 5 | `gt a b c` | `a = (b > c)` | 4 / 8 |
| 6 | `jmp a` | 跳到 a | 2 / 4 |
| 7 | `jt a b` | a 非零时跳到 b | 3 / 6 |
| 8 | `jf a b` | a 为零时跳到 b | 3 / 6 |
| 9 | `add a b c` | `(b + c) mod 32768` | 4 / 8 |
| 10 | `mult a b c` | `(b * c) mod 32768` | 4 / 8 |
| 11 | `mod a b c` | `b % c` | 4 / 8 |
| 12 | `and a b c` | `b & c` | 4 / 8 |
| 13 | `or a b c` | `b | c` | 4 / 8 |
| 14 | `not a b` | 15 位按位取反 | 3 / 6 |
| 15 | `rmem a b` | `a = memory[b]` | 3 / 6 |
| 16 | `wmem a b` | `memory[a] = b` | 3 / 6 |
| 17 | `call a` | 保存返回地址并跳到 a | 2 / 4 |
| 18 | `ret` | 弹出地址并跳转 | 1 / 2 |
| 19 | `out a` | 输出字符 | 2 / 4 |
| 20 | `in a` | 输入字符到 a | 2 / 4 |
| 21 | `noop` | 不做操作 | 1 / 2 |

表中 `a/b/c` 是否必须为寄存器取决于它是否是写入目标。`set` 的 a、`eq/gt/add/...` 的 a、`rmem` 的 a 和 `in` 的 a 必须是寄存器；其余读取位置可为立即数或寄存器编码。

## 4. 两套地址单位

这是本项目最关键的知识点。

Synacor ISA 使用 word 地址：

```text
地址 0 = 第 0 个 16 位 word
地址 1 = 第 1 个 16 位 word
```

GNU sim core 使用 byte 地址：

```text
Synacor word 0 -> sim byte 0
Synacor word 1 -> sim byte 2
Synacor word 2 -> sim byte 4
```

因此：

```c
synacor_address << 1  /* word 地址变 byte 地址 */
byte_address >> 1     /* byte 地址变 word 地址 */
```

普通 PC 更新也按 byte 计算：

```text
HALT/RET/NOOP       1 word -> 2 bytes
PUSH/POP/JMP/...    2 words -> 4 bytes
SET/JT/JF/...       3 words -> 6 bytes
EQ/ADD/MULT/...     4 words -> 8 bytes
```

## 5. 手工执行 README 示例

程序 word：

```text
9, 32768, 32769, 4, 19, 32768
```

布局：

| word 地址 | byte 地址 | 原始值 | 含义 |
|---:|---:|---:|---|
| 0 | 0 | 9 | ADD |
| 1 | 2 | 32768 | 目标 R0 |
| 2 | 4 | 32769 | 读取 R1 |
| 3 | 6 | 4 | 立即数 4 |
| 4 | 8 | 19 | OUT |
| 5 | 10 | 32768 | 读取 R0 |

假设开始时 `R1 = 61`：

```text
PC(byte)=0
ADD R0,R1,4 -> R0=65
PC(byte)=8
OUT R0       -> 输出 ASCII 65，即 A
PC(byte)=12
```

程序在 byte 12 之后还需要有 HALT，否则 CPU 会继续把后面的内存当指令执行。规范中的短示例主要用于解释两条指令，并不是一个完整、安全终止的程序。

## 6. CALL/RET 地址推演

假设 `CALL 10` 位于 Synacor word 地址 3，也就是 byte 地址 6：

```text
CALL 长 2 words = 4 bytes
下一条指令 byte 地址 = 6 + 4 = 10
保存到栈的 Synacor 地址 = 10 >> 1 = 5
目标 byte 地址 = 10 << 1 = 20
```

所以实现为：

```c
push ((pc + 4) >> 1);
pc = target << 1;
```

`RET` 弹出 word 地址 5，再执行 `5 << 1`，回到 byte 地址 10。
