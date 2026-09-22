# GNU sim `example-synacor` 学习笔记

这套笔记对应 `sim/example-synacor`，目标不是只读懂 22 条指令，而是借这个最小实现理解一个 GNU simulator 端口由哪些部分组成、公共框架如何调用架构代码，以及怎样为新架构搭出第一个可运行的模拟器。

文档全部使用相对路径。把整个 `binutils-gdb` 仓库复制到另一台电脑后，链接仍然有效；即使只复制 `My_learn`，正文也包含了继续学习所需的主要信息。

## 建议阅读顺序

1. [01-项目地图.md](01-项目地图.md)：先建立全局认识。
2. [02-ISA与二进制编码.md](02-ISA与二进制编码.md)：理解机器究竟执行什么。
3. [03-GNU-sim运行链路.md](03-GNU-sim运行链路.md)：理解公共框架怎样进入架构代码。
4. [04-step_once逐层精读.md](04-step_once逐层精读.md)：逐类阅读指令实现。
5. [05-构建测试与跟踪.md](05-构建测试与跟踪.md)：实际编译、运行和观察。
6. [06-源码缺口与练习.md](06-源码缺口与练习.md)：通过补测试和修正边界行为巩固理解。

## 最重要的心智模型

```text
独立前端 common/nrun.c
        |
        v
sim_open            创建 SIM_DESC、CPU、模块与内存
        |
        v
sim_load            通过 BFD 把 ELF 节加载进模拟内存
        |
        v
sim_create_inferior 设置入口 PC、argv、envp
        |
        v
sim_resume          GNU sim 通用的恢复/单步层
        |
        v
sim_engine_run      本端口的执行循环
        |
        v
step_once           取指、译码、执行、提交 PC
        |
        v
sim_engine_halt     HALT、异常、断点或单步停止
```

GNU sim 公共层负责：

- BFD 程序加载；
- 模拟内存和访问检查；
- 命令行选项；
- host callback 与终端 I/O；
- trace、profile 和事件调度；
- 单步与停止原因；
- 独立 `run` 前端以及供 GDB 调用的接口。

`example-synacor` 架构层负责：

- 8 个寄存器、PC 和内部 SP；
- Synacor 数字/寄存器编码的解释；
- 22 条 opcode 的语义；
- word 地址与 GNU sim byte 地址之间的换算。

## 快速自测

读完整套笔记后，应能不看答案解释以下问题：

1. 为什么普通指令按 `pc += 2/4/6/8` 前进，而跳转地址要 `<< 1`？
2. 为什么目标操作数使用 `register_num`，源操作数使用 `interp_num`？
3. `SIM_DESC`、`SIM_CPU` 和 `struct example_sim_cpu` 分别是什么层次？
4. `CALL` 为什么把 `(pc + 4) >> 1` 压栈？
5. `sim_engine_halt` 为什么能离开一个没有 `break` 的无限循环？
6. testsuite 为什么能用主机 assembler 测试一个根本没有 GAS 后端的假 ISA？

## 原始资料入口

- [项目说明](../example-synacor/README)
- [架构规范（含当前中文翻译）](../example-synacor/README.arch-spec)
- [GNU sim hacking notes](../README-HACKING)
- [模拟器测试目录](../testsuite/example-synacor/)
