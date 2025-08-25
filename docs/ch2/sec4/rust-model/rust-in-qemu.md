Rust in QEMU 是一个旨在支持使用 Rust 编程语言为 QEMU 添加新功能的项目。集成 Rust 的思路是通过 FFI 编程结合 Rust 的 bingen
工具来允许在 Rust 中调用 C 的接口，最后可以将 Rust 编译为静态库/动态库被 QEMU 项目链接，从而允许在 QEMU 运行时使用 Rust 编写的代码。

目前，主要聚焦在使用安全的 Rust 编写从 `SysBusDevice` 继承的设备。

!!! note

    后续计划支持编写其他类型的设备（例如支持 DMA 的 PCI 设备）、完整的主板或后端（例如块设备格式）。

    更多介绍详见官方文档：[Rust in QEMU][1]

## QEMU Rust crate

截止到 QEMU 10.1 版本，QEMU 中已经支持了 4 个 crate:

- qemu_api 用于与 C 代码的绑定以及提供有用功能

- qemu_api_macros 定义了几个在编写 C 代码时很有用的过程宏

- pl011 用于实现 QEMU 中的 PL011 串口设备

- hpet 用于实现 QEMU 中的 HPET 定时器设备

!!! note

    截止到 QEMU 10.1 版本，pl011 crate 与 hw/char/pl011.c 在提交 3e0f118f82 时保持同步。hpet crate 在提交 1433e38cc8 时保持同步。两者都缺少跟踪功能。

我们比较关心 qemu_api crate，因为它直接关系到我们如何使用 Rust 进行建模。

qemu_api 中又包含了许多 module，每个 module 对应了 QEMU 中的一个功能模块。

qemu_api 的模块的进展可定义为：

- complete: 可用于新设备；如适用，该 API 支持 C 语言中可用的全部功能

- stable: 可投入生产使用，该 API 安全可靠，不应发生重大变更

- proof of concept: 该 API 可能会发生变化，但允许使用安全的 Rust 语言

- initial: 该 API 处于初始阶段；它需要大量不安全的代码；可能存在健全性或类型安全问题

目前处于 complete 阶段的模块有：`bitops`、`callbacks`、`errno`、`irq`、`module`，
大家可以放心使用。

!!! note

    API 的稳定性并非承诺，因为 QEMU 的 C API 本身也不是稳定的接口。此外，unsafe 的接口日后可能会被安全的接口取代。


[1]: https://www.qemu.org/docs/master/devel/rust.html#rust-in-qemu