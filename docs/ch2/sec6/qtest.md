## 基本介绍

为了方便验证 QEMU 的设备模型，QEMU 支持了一个设备仿真测试框架，叫 Qtest，它的特点是通过一个额外的进程来和 QEMU 进行交互，主要通过 QMP 进行通信。

所以使用 Qtest 编写测例时，不需要想传统的外设驱动测例那样开发，只需要基于 Qtest 提供的 API 即可方便的访问设备模型的地址空间，不依赖任何 Arch。

另外 Qtest 还可以通过一种特殊的“qtest”协议控制 QEMU 的某些方面（如虚拟时钟步进）。

QTest 测试用例可以通过以下方式执行：

```bash
make check-qtest
```

!!! note

    在 libqtest 的基础上，还构建了一个更高级的库——libqos，用于封装设备驱动程序的常见任务，例如内存管理以及与系统总线或设备的通信。

    许多虚拟设备测试使用 libqos，而非直接调用 libqtest。libqos 还提供了Qgraph API，以提高每个测试的覆盖率，并自动完成 QEMU 命令行参数和设备设置。

    但本章节主要讲解 Qtest 的基本用法。

## 添加新测例

添加新 QTest 用例的步骤如下（官方文档翻译润色）：

1. 为测试创建一个新的源文件。（如有必要，可以添加多个文件。）例如，tests/qtest/foo-test.c。

2. 使用 glib 和 libqtest/libqos API 编写测试代码。也可以参考现有的测试和库头文件。

3. 在 tests/qtest/meson.build 中注册新测试。将测试可执行文件名称添加到适当 的 qtests_* 变量中。每个架构对应一个变量，此外还有 qtests_generic 用于可在所有架构上运行的测试。例如：

    ```
    qtests_generic = [
        ...
        'foo-test',
        ...
    ]
    ```

4. 如果测试有多个源文件，或者需要链接除 qemuutil 和 qos 之外的任何依赖项，请在 qtests 字典中列出它们。例如，需要使用 QIO 库的测试会有如下条目：

    ```
    {
      ...
      'foo-test': [io],
      ...
    }
    ```

调试 QTest 失败比调试单元测试稍微困难一些，因为这些测试会在环境变量中查找 QEMU 程序名称，例如 QTEST_QEMU_BINARY 和 QTEST_QEMU_IMG，而且也因为不容易将 gdb 附加到从测试中启动的 QEMU 进程上。但是，手动调用测试并在测试上使用 gdb 仍然很简单：从输出中找出实际的命令：

```
make check-qtest V=1
```

!!! note

    有关 Qtest 协议的内容，请阅读官方文档：[qtest][1]

[1]: https://www.qemu.org/docs/master/devel/testing/qtest.html

