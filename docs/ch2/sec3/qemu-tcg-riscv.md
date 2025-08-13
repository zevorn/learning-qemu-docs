## 基本介绍
QEMU 对 RISC-V 架构的模拟，仍然在 QOM 的框架下实现，可以支持不同的加速器（KVM、TCG...）。同时，为了方便管理各种 RISC-V 指令扩展，QEMU 设计了一个 RISCVCPUConfig 数据结构，允许为不同类型的 CPU，绑定对应的扩展，极大地的提高了灵活性和可维护性。

下面给出一个 QOM 与 RISC-V 的关系图（摘自 [PLCT Lab · 从零开始的 RISC-V 模拟器开发][1]）

```bash
                                                                                   +--------------------------+
                                                                                   | TYPE_RISCV_CPU_MAX       |
                                                                                   +--------------------------+
  cpu base type                                                                    | TYPE_RISCV_CPU_BASE32    |
+---------------+    +---------------+   +-------------+     +----------------+    +--------------------------+
|  TYPE_OBJECT  +--->|  TYPE_DEVICE  +-->|   TYPE_CPU  +---->| TYPE_RISCV_CPU +--> | TYPE_RISCV_CPU_SHAKTI_C  |
+---------------+    +---------------+   +-------------+     +----------------+    +--------------------------+
                                                                                   | TYPE_RISCV_CPU_VEYRON_V1 |
  cpu class (interface)                                                            +--------------------------+
+---------------+    +---------------+   +-------------+     +----------------+    | TYPE_RISCV_CPU_HOST      |
|  ObjectClass  +--->|  DeviceClass  +-->|   CPUClass  +---->| RISCVCPUClass  |    +--------------------------+
+---------------+    +---------------+   +-------------+     +----------------+                                
                                                                                                               
  cpu object (property)
+---------------+    +---------------+   +-------------+   +-------------------+                               
|  Object       +--->|  DeviceState  +-->|   CPUState  +-->|  RISCVCPU(ArchCPU)|                               
+---------------+    +---------------+   +-------------+   +-------------------+                               
```

## 添加新 CPU 类型

[待补充]


## 添加指令集扩展
[待补充]


[1]: https://github.com/plctlab/writing-your-first-riscv-simulator/blob/main/S01E07-CPU-Simulation-Part1-in-Qemu.pdf
