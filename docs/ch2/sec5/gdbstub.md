
!!! note "推荐阅读"

    [GDB 调试 QEMU 上运行的 Linux 内核][1]

QEMU 支持远程调试 user 模式和 system 模式下的客户机程序。QEMU 内置了一个 gdbserver 可以对客户机处理器进行控制。

用户可以通过任何 gdb client 连接。

下面给出具体操作命令：

```bash
$QEMU $QEMU_ARGS -s -S
```

- `-s`：启动 gdbstub，并把端口号设置为 1234

- `-S`：让 QEMU 停在客户机的第一条指令，等待 gdb client 连接。

如果要指定调试的端口号，可以使用如下命令：

```bash
$QEMU $QEMU_ARGS -gdb tcp::<your-port> -S
```

然后使用对应架构的 gdb 连接 QEMU gdbstub：

```bash
$ARCH-gdb $BINARY -ex "target remote localhost:1234"
```

[1]: https://www.qemu.org/docs/master/system/gdb.html
