# 03 GNU sim 运行链路

## 1. 独立运行器从哪里开始

`example-synacor/run` 的 `main()` 不在架构目录，而来自 [`common/nrun.c`](../common/nrun.c)。主要调用顺序是：

```text
default_callback.init
sim_open
bfd_openr / sim_analyze_program
sim_load
sim_create_inferior
sim_resume
sim_stop_reason
sim_close
```

这说明架构端口不需要自己实现完整命令行程序。它向框架提供约定接口，公共前端负责组织生命周期。

## 2. `sim_open`：创建模拟器

[`example-synacor/interp.c`](../example-synacor/interp.c) 中的 `sim_open` 可以分成七步。

### 2.1 分配全局状态

```c
SIM_DESC sd = sim_state_alloc (kind, callback);
```

`kind` 表示独立运行或由调试器打开；`callback` 抽象宿主 I/O 和文件操作。

### 2.2 设置目标基本属性

```c
current_alignment = STRICT_ALIGNMENT;
current_target_byte_order = BFD_ENDIAN_LITTLE;
```

Synacor word 必须对齐，并以小端序存储。

### 2.3 分配架构私有 CPU 数据

```c
sim_cpu_alloc_all_extra(sd, 0, sizeof(struct example_sim_cpu));
```

公共 `SIM_CPU` 与私有 `example_sim_cpu` 分开分配，通过 `CPU_ARCH_DATA` 相连。

### 2.4 初始化并解析公共选项

```text
sim_pre_argv_init
sim_parse_args
sim_analyze_program
sim_config
sim_post_argv_init
```

这条公共流水线建立模块、解析诸如 `--trace-insn` 的选项、分析 BFD 程序并完成配置。

任何一步失败都会调用 `free_state`，依次卸载模块、释放 CPU 和全局状态。

### 2.5 初始化 CPU

```c
initialize_cpu(sd, cpu);
```

该函数：

- 清零 R0～R7；
- 把 PC 设为 0；
- 把内部 SP 设为 `0x80000`；
- 注册 PC 的读写回调。

```c
CPU_PC_FETCH(cpu) = pc_get;
CPU_PC_STORE(cpu) = pc_set;
```

公共框架以后通过 `sim_pc_get/sim_pc_set` 访问 PC，而无需知道 PC 存在私有结构的哪个字段。

### 2.6 分配模拟内存

代码先探测地址 4 是否已映射；如果没有，则执行：

```text
memory-size 0x1000000
```

也就是默认分配 16 MiB。ISA 可见内存只占前 65536 bytes，高地址空间被实现拿来保存逻辑栈。

## 3. `sim_load`：装载程序

公共 `nrun.c` 调用 `sim_load`，通过 BFD 将可装载 section 写进 sim core memory。

需要注意：独立 `run` 通常期待 BFD 能识别的 object，例如 ELF。Synacor 的原始裸二进制并不会因为后缀是 `.bin` 就自动拥有 section、VMA 和入口地址。testsuite 的做法是使用普通 GNU assembler 的 `.byte` 生成 ELF。

## 4. `sim_create_inferior`：准备一次执行

该函数主要做两件事。

第一，设置入口 PC：

```c
addr = abfd ? bfd_get_start_address(abfd) : 0;
sim_pc_set(cpu, addr);
```

第二，保存 argv/envp 并交给 callback。独立 `run` 的参数通常已经在 `sim_open -> sim_parse_args` 期间建立；GDB 则可以在每次 `run` 前换一组参数，因此这里仍需同步。

注意：`sim_create_inferior` 只显式重设 PC，并未再次清零通用寄存器和 SP。`initialize_cpu` 才像上电复位，并且在这个实现里发生于 `sim_open`。

## 5. `sim_resume`：单步和停止的桥梁

[`common/sim-resume.c`](../common/sim-resume.c) 是通用实现。

若调用者要求单步，它向 event engine 安排一个在 1 tick 后发生的事件：

```c
engine->stepper = sim_events_schedule(sd, 1, has_stepped, sd);
```

随后保存 `jmp_buf` 并进入架构的 `sim_engine_run`。`sim_engine_halt` 会把退出原因写入 engine，然后借助 `longjmp` 回到 `sim_resume`。所以架构执行循环可以写成无限循环，不必让每层函数返回特殊状态码。

## 6. `sim_engine_run`：最小执行循环

本端口固定取第 0 颗 CPU：

```c
cpu = STATE_CPU(sd, 0);
```

然后循环：

```c
while (1) {
  step_once(cpu);
  if (sim_events_tick(sd))
    sim_events_process(sd);
}
```

一条指令算一个 tick。事件机制使以下行为能够发生在指令边界：

- 单步停止；
- 外部停止请求；
- 其他被调度的 simulator 事件。

## 7. 三种重要停止语义

典型调用形式：

```c
sim_engine_halt(sd, cpu, NULL, pc, reason, signal_or_status);
```

本项目主要使用：

```text
sim_exited, 0             HALT 或扩展的 Q 退出，进程成功
sim_signalled, SIM_SIGILL 非法 opcode/操作数
sim_stopped, SIM_SIGTRAP  通用单步事件触发
```

`nrun.c` 最后用 `sim_stop_reason` 取出原因，并将退出码或宿主 signal 映射为运行器的返回值。

## 8. 为什么 I/O 通过 callback

OUT 调用：

```c
sim_io_printf(sd, "%c", value);
```

IN 调用：

```c
sim_io_read_stdin(sd, &c, 1);
```

不用裸 `printf/getchar`，是为了让同一模拟器既能在独立进程中工作，也能嵌入 GDB 或由其他 host callback 接管 I/O。
