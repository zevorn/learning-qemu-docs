G233 是为 QEMU 训练营定制的虚拟板卡，具体介绍如下：

板卡示意图：

```
+------------------------------------------------------------------------------+
|                       G233 Board Architecture Diagram                        |
|                                                                              |
|  +-----------------+       +--------------------+       +-----------------+  |
|  |       DRAM      | <---> |   riscv.g233.cpu   | <---> |       MROM      |  |
|  |   0x8000_0000   |       | - RVA23            |       |   0x0000_1000   |  |
|  |   0xBFFF_FFFF   |       | - Other Extensions |       |   0x0000_2FFF   |  |
|  +-----------------+       +----------+---------+       +-----------------+  |
|                                       |                                      |
|                   +-------------------+-------------------+                  |
|                   |                   |                   |                  |
|           +-------v-------+   +-------v-------+   +-------v-------+          |
|           |      PLIC     |   |     CLINT     |   |     PL011     |          |
|           |  0x0C00_0000  |   |  0x0200_0000  |   |  0x1000_0000  |          |
|           |  0x0FFF_FFFF  |   |  0x0200_BFFF  |   |  0x1000_0FFF  |          |
|           +--------+------+   +--------+------+   +---------------+          |
|                    |                   |                                     |
|           +--------v------+   +--------v------+   +---------------+          |
|           |      GPIO     |   |      PWM0     |   |               |          |
|           |  0x1001_2000  |   |  0x1001_5000  |   |     Other     |          |
|           |  0x1001_20FF  |   |  0x1001_5FFF  |   |               |          |
|           +---------------+   +---------------+   +---------------+          |
|                                                                              |
|                              Bus Fabric / AXI/LIB                            |
|                                                                              |
+------------------------------------------------------------------------------+
```

板卡基本参数：

|      address space      |         name         |  desc  |
|           :---:         |         ----         |  ----  |
|             -           | riscv.g233.cpu      |  RISC-V G233 CPU 核心<br>符合 RISC-V ISA RVA23 标准，并扩展了自定义指令集，用于 QEMU 训练营教学实践。  |
| `0x00001000 0x00002fff` | riscv.g233.mrom      |  Machine Read-Only Memory (MROM)<br>存储芯片启动时的固件或引导代码（Boot ROM），内容在出厂时固化，不可修改，用于初始化 CPU 和加载下一阶段引导程序。  |
| `0x02000000 0x02003fff` | riscv.aclint.swi     |  ACLINT Software Interrupts (MSWI)<br>用于核间通信和软件触发的中断，支持 RISC-V 的 Machine-mode 软件中断，常用于操作系统中核间调度或同步操作。 |
| `0x02004000 0x0200bfff` | riscv.aclint.mtimer  |  ACLINT Machine Timer (MTIME)<br>提供高精度定时器功能，用于实现 rdtime 指令、时间片调度、超时控制等，是 RISC-V 标准计时器模块（MTimer）的实现。 |
| `0x0c000000 0x0fffffff` | riscv.sifive.plic    |  SiFive Platform-Level Interrupt Controller (PLIC)<br>管理外部中断的优先级和分发，支持多个中断源和多个中断优先级，负责将外部设备中断路由到合适的 CPU 核心。  |
| `0x10000000 0x10000fff` | riscv.pl011          |  PL011 UART 控制器<br>兼容 ARM PL011 的串行通信接口，用于调试输出、终端通信等，常用于打印日志和交互式调试。  |
| `0x10012000 0x100120ff` | sifive_soc.gpio      |  General Purpose Input/Output (GPIO)<br>通用输入输出接口，可用于控制 LED、按键、传感器等外设，支持配置引脚方向、电平读写等基本操作。  |
| `0x10015000 0x10015fff` | riscv.g233.pwm0      |  Pulse Width Modulation (PWM) 控制器<br>用于生成可调占空比的方波信号，常用于控制电机速度、LED 亮度调节、电源管理等模拟信号控制场景。   |
| `0x80000000 0xbfffffff` | riscv.g233.ram       |  Main DRAM (Dynamic Random Access Memory)<br>主内存区域，用于运行操作系统、应用程序和存储运行时数据，是系统主要的可读写存储空间。  |

