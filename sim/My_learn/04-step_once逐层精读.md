# 04 `step_once` 逐层精读

## 1. 函数骨架

`step_once` 的稳定结构是：

```text
1. 从 SIM_CPU 取得 sd 和架构私有状态
2. 通过 sim_pc_get 取得当前 byte PC
3. 从 exec_map 读取 16 位 opcode
4. 读取并解释操作数
5. 执行指令语义
6. 顺序增加 PC，或把 PC 替换为跳转目标
7. 通过 sim_pc_set 提交新 PC
```

开头的对象关系：

```c
SIM_DESC sd = CPU_STATE(cpu);
struct example_sim_cpu *example_cpu = EXAMPLE_SIM_CPU(cpu);
sim_cia pc = sim_pc_get(cpu);
```

取 opcode：

```c
iw1 = sim_core_read_aligned_2(cpu, pc, exec_map, pc);
```

参数意义可按以下方式理解：

```text
cpu       哪颗 CPU 发起访问
pc        当前指令地址，用于报告 fault
exec_map  这是取指访问
pc        实际读取地址
```

读取数据用 `read_map`，写入数据用 `write_map`。

## 2. 原始 word 与解释值

变量命名体现了两层概念：

```text
iw1/iw2/...   instruction word，内存中的原始 16 位编码
num1/num2/... 解释后的 opcode、寄存器编号或实际数值
```

例如 `ADD R0,R1,4`：

```text
iw2 = 32768, num2 = 0       目标寄存器编号
iw3 = 32769, num3 = regs[1] 源值
iw4 = 4,     num4 = 4       源值
```

## 3. 简单控制类

### HALT

```c
sim_engine_halt(sd, cpu, NULL, pc, sim_exited, 0);
```

不会正常返回到函数末尾，因而无需更新 PC。

### NOOP

不修改任何架构状态，只执行 `pc += 2`。

### 非法 opcode

最后的 `else` 用 `SIM_SIGILL` 停止。这与正常 HALT 的 `sim_exited` 不同，独立运行器会把它视为失败。

## 4. 数据移动类

### SET

目标用 `register_num`，源用 `interp_num`：

```c
regs[target_reg] = source_value;
pc += 6;
```

### PUSH/POP

初始：

```text
SP = 0x80000
```

PUSH：

```text
memory[SP] = value
SP -= 2
```

POP：

```text
SP += 2
value = memory[SP]
target_register = value
```

例如连续压入 1、3：

```text
写 [0x80000] = 1, SP=0x7fffe
写 [0x7fffe] = 3, SP=0x7fffc
POP -> SP=0x7fffe, 读出 3
POP -> SP=0x80000, 读出 1
```

这是“SP 指向下一个空位置”的向下生长栈变体。

## 5. 比较和条件分支

EQ/GT 将 C 关系表达式结果写入寄存器。C 中关系表达式恰好产生 0 或 1，符合 ISA。

JT/JF 的目标首先通过 `interp_num` 取得 Synacor word 地址，然后 `<< 1` 变为 sim byte 地址。

```text
JT: condition != 0 时 taken
JF: condition == 0 时 taken
```

未跳转时 `pc += 6`；已跳转时直接替换 PC，不能再增加顺序长度。

## 6. 算术与位运算

### ADD/MULT

```c
result = operation % 32768;
```

输入最大为 32767。乘法最大约为 10.7 亿，小于常见 32 位有符号整数上限；C 的整数提升使两个 `uint16_t` 通常先提升为 `int`，这里不会溢出。

### MOD

```c
result = num3 % num4;
```

实现未检查 `num4 == 0`，这是后续练习中的边界问题。

### AND/OR

两个输入本来就限制在 15 位，所以结果仍是 15 位。

### NOT

C 的 `~` 会对提升后的完整整数取反，因此必须重新掩码：

```c
result = (~num3) & 0x7fff;
```

否则高位会成为 1，不再是有效 Synacor 值。

## 7. 内存类

RMEM：

```text
读取源操作数 b
b << 1 变成 byte 地址
从 read_map 读取 16 位值
写入目标寄存器 a
```

WMEM：

```text
读取目标地址 a 和数据 b
a << 1 变成 byte 地址
向 write_map 写入 16 位值
```

Synacor 允许修改程序所在的同一片内存，因此 `mem.s` 测试能够改写开头的 JMP，之后重新跳回地址 0 执行新指令。这属于 self-modifying code。

## 8. CALL/RET

CALL 的两个地址单位不同：

```c
sim_core_write_aligned_2(..., sp, (pc + 4) >> 1);
pc = target << 1;
```

- GNU sim 内部 PC 是 byte 地址；
- ISA 栈中保存的是 Synacor word 地址；
- CALL 自身有 2 个 word，即 4 bytes。

RET 弹出 word 地址后再 `<< 1` 恢复为 byte PC。

## 9. IN/OUT

OUT 将源值作为字符交给 `sim_io_printf`。

IN 的目标必须是寄存器。它一次读取一个宿主字符，并包含非 ISA 扩展：输入大写 `Q` 立即正常退出。代码使用 `char c`；如果宿主的 `char` 为 signed 且输入字节大于 127，转存到 `uint16_t` 时可能得到非 15 位值。Challenge 假定小写文本输入，因此正常任务不会触发。

## 10. TRACE 宏的层次

源码有意展示不同 trace 类别：

| 宏 | 观察内容 |
|---|---|
| `TRACE_EXTRACT` | 从指令流取到了哪些原始 word |
| `TRACE_DECODE` | 原始编码如何解释、条件结果如何计算 |
| `TRACE_INSN` | 人类可读的当前指令 |
| `TRACE_REGISTER` | 寄存器和 PC/SP 修改 |
| `TRACE_MEMORY` | 显式数据内存读写 |
| `TRACE_BRANCH` | 跳转、调用、返回 |
| `TRACE_EVENTS` | 终端 I/O 等事件 |

这些宏是否输出由运行参数和编译配置决定，关闭时不应改变架构语义。

## 11. 一条 ADD 的完整状态变化

初始状态：

```text
PC(byte)=0
R1=32766
memory words=[9,32768,32769,4,...]
```

执行：

```text
fetch word at byte 0 -> opcode 9
fetch word at byte 2 -> 32768 -> target R0
fetch word at byte 4 -> 32769 -> value R1=32766
fetch word at byte 6 -> 4 -> literal 4
result=(32766+4)%32768=2
R0=2
PC=0+8=8
```

函数末尾统一调用 `sim_pc_set(cpu, pc)` 提交 PC。
