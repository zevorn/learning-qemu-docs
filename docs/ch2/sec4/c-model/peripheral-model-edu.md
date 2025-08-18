在计算机系统中，CPU 与外设的交互本质是 地址访问 和 中断通知：

- 地址访问：CPU 通过 MMIO（内存映射 I/O）或 PIO（端口 I/O）读写设备寄存器

- 中断通知：外设通过中断控制器向 CPU 发送事件信号

下面给出一个伪代码：

```
// 伪代码：CPU 与设备交互流程
while (1) {
    if (中断触发) {
        读取设备状态寄存器;
        处理数据;
    }
    写入控制寄存器启动操作;
}
```
下面我们以 edu 设备为例进行外设建模的讲解。

## edu 设备介绍

edu 是 QEMU 官方提供的教学用虚拟设备，其功能设计精简却完整：

- 基础功能：实现阶乘计算（fact 寄存器）

- 高级功能：DMA 传输、中断触发

- 典型应用：演示 PCI 设备全生命周期管理

## 定义设备属性

关键点：通过继承 PCIDevice 自动获得 PCI 配置空间管理能力。

```c
typedef struct EduState {
    PCIDevice pdev;          // 继承 PCI 设备基类
    MemoryRegion mmio;        // MMIO 内存区域
    uint32_t fact;            
    uint32_t status;          // 状态寄存器 (bit0=计算中，bit7=中断使能)
    QemuThread thread;        // 后台计算线程
    QemuMutex thr_mutex;      // 线程锁
    bool stopping;            // 线程停止标志
} EduState;
```

## 初始化 PCI 配置空间

在 pci_edu_realize() 中设置设备标识

```c
pci_config_set_vendor_id(pci_conf, 0x1234);  // 厂商 ID
pci_config_set_device_id(pci_conf, 0x11e8);  // 设备 ID
pci_config_set_class(pci_conf, PCI_CLASS_OTHERS); // 设备类
```

此步骤让操作系统能识别设备为 "Edu Device"

## 映射 MMIO 区域

```c
memory_region_init_io(&edu->mmio, OBJECT(edu), &edu_mmio_ops, edu,
                      "edu-mmio", 0x1000); // 4KB 地址空间
pci_register_bar(pdev, 0, PCI_BASE_ADDRESS_SPACE_MEM, &edu->mmio);
```

地址布局：

```bash
0x00 : 阶乘输入寄存器 (可写)
0x04 : 阶乘结果寄存器 (只读)
0x80 : DMA 源地址寄存器
0x88 : DMA 目标地址寄存器
```

## 实现 MMIO 回调函数

以阶乘计算为例：

```c
static uint64_t edu_mmio_read(void *opaque, hwaddr addr, unsigned size)
{
    EduState *edu = opaque;
    if (addr == 0x04) { // 读取结果寄存器
        return edu->fact;
    }
    ...
}

static void edu_mmio_write(void *opaque, hwaddr addr, uint64_t val, unsigned size)
{
    EduState *edu = opaque;
    if (addr == 0x00) { // 写入输入值
        qemu_mutex_lock(&edu->thr_mutex);
        edu->status |= EDU_STATUS_COMPUTING; // 标记计算中
        qemu_cond_signal(&edu->thr_cond);    // 唤醒计算线程
        qemu_mutex_unlock(&edu->thr_mutex);
    }
}
```

## 实现后台计算线程

关键设计：避免阻塞主线程，通过条件变量实现异步计算。

```c
static void *edu_fact_thread(void *opaque)
{
    EduState *edu = opaque;
    while (!edu->stopping) {
        qemu_mutex_lock(&edu->thr_mutex);
        while (!(edu->status & EDU_STATUS_COMPUTING)) {
            qemu_cond_wait(&edu->thr_cond, &edu->thr_mutex);
        }
        uint32_t input = ...; // 从寄存器读取输入值
        uint32_t result = compute_factorial(input); // 计算阶乘
        edu->fact = result;   // 写入结果寄存器
        edu->status &= ~EDU_STATUS_COMPUTING; // 清除计算标志
        if (edu->status & EDU_STATUS_IRQFACT) {
            edu_raise_irq(edu, FACT_IRQ); // 触发中断
        }
        qemu_mutex_unlock(&edu->thr_mutex);
    }
    return NULL;
}
```

## 中断触发机制

```c
void edu_raise_irq(EduState *edu, uint32_t val)
{
    if (msi_enabled(&edu->pdev)) { // 使用 MSI 中断
        msi_notify(&edu->pdev, val);
    } else { // 使用传统 INTx 中断
        pci_irq_assert(&edu->pdev);
    }
}
```

## DMA 传输实现

```c
static void edu_dma_timer(EduState *edu)
{
    dma_addr_t src = edu->dma.src;
    dma_addr_t dst = edu->dma.dst;
    uint32_t cnt = edu->dma.cnt;
    
    // 执行 DMA 拷贝
    pci_dma_read(&edu->pdev, src, edu->dma_buf, cnt);
    pci_dma_write(&edu->pdev, dst, edu->dma_buf, cnt);
    
    // 触发中断
    if (edu->dma.cmd & EDU_DMA_IRQ) {
        edu_raise_irq(edu, DMA_IRQ);
    }
}
```

## 客户机视角的设备交互

当客户机程序的驱动访问 edu 设备时：

1. 探测设备：通过 PCI ID 0x1234:11e8 识别

2. 映射 MMIO：ioremap() 获取寄存器虚拟地址

3. 计算阶乘

4. 处理中断：在中断服务例程中清除状态标志

## 要点总结

edu 浓缩了外设建模的 5 大核心要素：

```
寄存器操作 → 中断通知 → DMA 传输 → 多线程协同 → PCI 规范
```

主要涉及：

- 状态机模型：通过寄存器位表示设备状态（如计算中/完成）

- 异步处理：耗时操作移交后台线程，避免阻塞 VCPU

- 地址隔离：每个设备拥有独立 MMIO 空间

- 中断抽象：支持传统 INTx 和现代 MSI 两种模式

- DMA 安全：地址校验防止虚拟机逃逸
