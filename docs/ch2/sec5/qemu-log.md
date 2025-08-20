QEMU 有一个灵活的日志系统，可以很方便地观测客户机的各种状态（指令流、中断、异常、系统调用）。

下面给出基本命令格式：

```bash
$QEMU $QEMU_ARGS -d <log-type,...> -D <log-file-name>
```

- `-d`：指定 log 的类型，可以多个，使用 `,` 分割

- `-D`：指定输出 log 的文件路径，如果不加这个参数，默认输出到命令行终端

你可以使用如下命令查看当前支持的 log 的类型：

```bash
$QEMU -d ?
Log items (comma separated):
out_asm         show generated host assembly code for each compiled TB
in_asm          show target assembly code for each compiled TB
op              show micro ops for each compiled TB
op_opt          show micro ops after optimization
op_ind          show micro ops before indirect lowering
op_plugin       show micro ops before plugin injection
int             show interrupts/exceptions in short format
exec            show trace before each executed TB (lots of logs)
cpu             show CPU registers before entering a TB (lots of logs)
fpu             include FPU registers in the 'cpu' logging
mmu             log MMU-related activities
pcall           x86 only: show protected mode far calls/returns/exceptions
cpu_reset       show CPU state before CPU resets
unimp           log unimplemented functionality
guest_errors    log when the guest OS does something invalid (eg accessing a
non-existent register)
page            dump pages at beginning of user mode emulation
nochain         do not chain compiled TBs so that "exec" and "cpu" show
complete traces
plugin          output from TCG plugins
strace          log every user-mode syscall, its input, and its result
tid             open a separate log file per thread; filename must contain '%d'
vpu             include VPU registers in the 'cpu' logging
invalid_mem     log invalid memory accesses
trace:PATTERN   enable trace events

Use "-d trace:help" to get a list of trace events.
```

我们列举一些常用的组合。

如果我们想观察 TCG 是如何翻译指令，可以使用如下命令：

```bash
$QEMU $QEMU_ARGS -d in_asm,op,out_asm -D tcg.log
```

如果我们想观察 CPU 的状态（寄存器值，中断/异常），可以使用下面的命令：

```bash
$QEMU $QEMU_ARGS -d exec,cpu,int -D cpu.log
```

如果你想获取 CPU 精准执行的指令流，需要设置每个 TB 只包含一条指令，可以使用下面的命令：

```bash
$QEMU $QEMU_ARGS --accel tcg,one-insn-per-tb=on -d exec,cpu,int -D cpu.log
```
