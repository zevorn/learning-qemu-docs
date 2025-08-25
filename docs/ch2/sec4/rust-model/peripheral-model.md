使用 Rust 建模的流程与 C 的区别不大，只不过需要遵循 Rust 的规范，另外需要处理 FFI 相关的内容。

我们仍然从 QOM 的角度出发，将创建一个 Rust 版本的外设模型分为以下几个部分：

- Class 的创建：方法、属性
- Object 的创建：初始化、实例化
- read/write 的实现：寄存器副作用

下面我们以 pl011 为例，进行讲解。

## Class 的创建

我们想给出 C 语言版本：

```c
static const TypeInfo pl011_arm_info = {
    .name          = TYPE_PL011,
    .parent        = TYPE_SYS_BUS_DEVICE,
    .instance_size = sizeof(PL011State),
    .instance_init = pl011_init,
    .class_init    = pl011_class_init,
};

static void pl011_luminary_init(Object *obj)
{
    PL011State *s = PL011(obj);

    s->id = pl011_id_luminary;
}

static const TypeInfo pl011_luminary_info = {
    .name          = TYPE_PL011_LUMINARY,
    .parent        = TYPE_PL011,
    .instance_init = pl011_luminary_init,
};

static void pl011_register_types(void)
{
    type_register_static(&pl011_arm_info);
    type_register_static(&pl011_luminary_info);
}

type_init(pl011_register_types)
```

与 C 定义 Typeinfo 类似，我们需要初始化这个 Class 的 type，指明它的 parent，并且注册 class_init 和 instance_init 相关的方法。


```rust

#[repr(C)]
pub struct PL011Class {
    parent_class: <SysBusDevice as ObjectType>::Class,
    /// The byte string that identifies the device.
    device_id: DeviceId,
}

trait PL011Impl: SysBusDeviceImpl + IsA<PL011State> {
    const DEVICE_ID: DeviceId;
}

impl PL011Class {
    fn class_init<T: PL011Impl>(&mut self) {
        self.device_id = T::DEVICE_ID;
        self.parent_class.class_init::<T>();
    }
}

unsafe impl ObjectType for PL011State {
    type Class = PL011Class;
    const TYPE_NAME: &'static CStr = crate::TYPE_PL011;
}

impl PL011Impl for PL011State {
    const DEVICE_ID: DeviceId = DeviceId(&[0x11, 0x10, 0x14, 0x00, 0x0d, 0xf0, 0x05, 0xb1]);
}

impl ObjectImpl for PL011State {
    type ParentType = SysBusDevice;

    const INSTANCE_INIT: Option<unsafe fn(ParentInit<Self>)> = Some(Self::init);
    const INSTANCE_POST_INIT: Option<fn(&Self)> = Some(Self::post_init);
    const CLASS_INIT: fn(&mut Self::Class) = Self::Class::class_init::<Self>;
}

impl DeviceImpl for PL011State {
    fn properties() -> &'static [Property] {
        &PL011_PROPERTIES
    }
    fn vmsd() -> Option<&'static VMStateDescription> {
        Some(&VMSTATE_PL011)
    }
    const REALIZE: Option<fn(&Self) -> qemu_api::Result<()>> = Some(Self::realize);
}

impl ResettablePhasesImpl for PL011State {
    const HOLD: Option<fn(&Self, ResetType)> = Some(Self::reset_hold);
}

impl SysBusDeviceImpl for PL011State {}

```

## Object 的创建

对于 C 来说，主要是定义 pl011 的 state 结构体，这一点在 Rust 当中是一样的。

我们先给出 C 的定义：

```c
struct PL011State {
    SysBusDevice parent_obj;

    MemoryRegion iomem;
    uint32_t flags;
    uint32_t lcr;
    uint32_t rsr;
    uint32_t cr;
    uint32_t dmacr;
    uint32_t int_enabled;
    uint32_t int_level;
    uint32_t read_fifo[PL011_FIFO_DEPTH];
    uint32_t ilpr;
    uint32_t ibrd;
    uint32_t fbrd;
    uint32_t ifl;
    int read_pos;
    int read_count;
    int read_trigger;
    CharBackend chr;
    qemu_irq irq[6];
    Clock *clk;
    bool migrate_clk;
    const unsigned char *id;
    /*
     * Since some users embed this struct directly, we must
     * ensure that the C struct is at least as big as the Rust one.
     */
    uint8_t padding_for_rust[16];
};

```

对应 Rust 的定义：

```rust
#[repr(C)]
#[derive(qemu_api_macros::Object)]
/// PL011 Device Model in QEMU
pub struct PL011State {
    pub parent_obj: ParentField<SysBusDevice>,
    pub iomem: MemoryRegion,
    #[doc(alias = "chr")]
    pub char_backend: CharBackend,
    pub regs: BqlRefCell<PL011Registers>,
    /// QEMU interrupts
    ///
    /// ```text
    ///  * sysbus MMIO region 0: device registers
    ///  * sysbus IRQ 0: `UARTINTR` (combined interrupt line)
    ///  * sysbus IRQ 1: `UARTRXINTR` (receive FIFO interrupt line)
    ///  * sysbus IRQ 2: `UARTTXINTR` (transmit FIFO interrupt line)
    ///  * sysbus IRQ 3: `UARTRTINTR` (receive timeout interrupt line)
    ///  * sysbus IRQ 4: `UARTMSINTR` (momem status interrupt line)
    ///  * sysbus IRQ 5: `UARTEINTR` (error interrupt line)
    /// ```
    #[doc(alias = "irq")]
    pub interrupts: [InterruptSource; IRQMASK.len()],
    #[doc(alias = "clk")]
    pub clock: Owned<Clock>,
    #[doc(alias = "migrate_clk")]
    pub migrate_clock: bool,
}

// Some C users of this device embed its state struct into their own
// structs, so the size of the Rust version must not be any larger
// than the size of the C one. If this assert triggers you need to
// expand the padding_for_rust[] array in the C PL011State struct.
static_assert!(size_of::<PL011State>() <= size_of::<qemu_api::bindings::PL011State>());

qom_isa!(PL011State : SysBusDevice, DeviceState, Object);

```

值得注意的是，C 当中进行类型转换是很方便的，但是在 Rust 中却不行。

因此 QEMU Rust 实现了一些 trait 和 宏，来安全的实现这一功能，对应 `qom_isa!`，我们简单看看它的实现：

首先是 `IsA<P> Trait`:

```rust
/// Marker trait: `Self` can be statically upcasted to `P` (i.e. `P` is a direct
/// or indirect parent of `Self`).
///
/// # Safety
///
/// The struct `Self` must be `#[repr(C)]` and must begin, directly or
/// indirectly, with a field of type `P`.  This ensures that invalid casts,
/// which rely on `IsA<>` for static checking, are rejected at compile time.
pub unsafe trait IsA<P: ObjectType>: ObjectType {}
```

这是一个标记性 trait（marker trait），表示类型 Self 可以“向上转型”为类型 P，即 P 是 Self 的父类（或祖先类）。

`unsafe trait` 表示实现这个 trait 是 unsafe 的，因为如果错误地实现，可能导致未定义行为（UB）。

然后是`IsA` 的自动实现（对自身）:

```rust
// SAFETY: it is always safe to cast to your own type
unsafe impl<T: ObjectType> IsA<T> for T {}
```

任何类型 T 都可以“向上转型”为自己（恒等转换），这是合理的，且是安全的，因为指针到自身的转换不会改变地址或破坏类型。

最后是 `qom_isa!` 宏的实现：

```rust
#[macro_export]
macro_rules! qom_isa {
    ($struct:ty : $($parent:ty),* ) => {
        $(
            // SAFETY: it is the caller responsibility to have $parent as the
            // first field
            unsafe impl $crate::qom::IsA<$parent> for $struct {}

            impl AsRef<$parent> for $struct {
                fn as_ref(&self) -> &$parent {
                    // SAFETY: follows the same rules as for IsA<U>, which is
                    // declared above.
                    let ptr: *const Self = self;
                    unsafe { &*ptr.cast::<$parent>() }
                }
            }
        )*
    };
}
```

这是个宏，用于方便地为某个结构体声明其“父类”。

当然，实现以上所有的前提是 `#[repr(C)]`, 它保证了结构体起始地址就是第一个字段的地址，与 C 是对齐的。

现在我们来看看初始化的过程：

```c

static void pl011_init(Object *obj)
{
    SysBusDevice *sbd = SYS_BUS_DEVICE(obj);
    PL011State *s = PL011(obj);
    int i;

    memory_region_init_io(&s->iomem, OBJECT(s), &pl011_ops, s, "pl011", 0x1000);
    sysbus_init_mmio(sbd, &s->iomem);
    for (i = 0; i < ARRAY_SIZE(s->irq); i++) {
        sysbus_init_irq(sbd, &s->irq[i]);
    }

    s->clk = qdev_init_clock_in(DEVICE(obj), "clk", pl011_clock_update, s,
                                ClockUpdate);

    s->id = pl011_id_arm;
}

static void pl011_realize(DeviceState *dev, Error **errp)
{
    PL011State *s = PL011(dev);

    qemu_chr_fe_set_handlers(&s->chr, pl011_can_receive, pl011_receive,
                             pl011_event, NULL, s, NULL, true);
}

```

对应 Rust 的实现：

```rust

impl PL011State {
    /// Initializes a pre-allocated, uninitialized instance of `PL011State`.
    ///
    /// # Safety
    ///
    /// `self` must point to a correctly sized and aligned location for the
    /// `PL011State` type. It must not be called more than once on the same
    /// location/instance. All its fields are expected to hold uninitialized
    /// values with the sole exception of `parent_obj`.
    unsafe fn init(mut this: ParentInit<Self>) {
        static PL011_OPS: MemoryRegionOps<PL011State> = MemoryRegionOpsBuilder::<PL011State>::new()
            .read(&PL011State::read)
            .write(&PL011State::write)
            .native_endian()
            .impl_sizes(4, 4)
            .build();

        // SAFETY: this and this.iomem are guaranteed to be valid at this point
        MemoryRegion::init_io(
            &mut uninit_field_mut!(*this, iomem),
            &PL011_OPS,
            "pl011",
            0x1000,
        );

        uninit_field_mut!(*this, regs).write(Default::default());

        let clock = DeviceState::init_clock_in(
            &mut this,
            "clk",
            &Self::clock_update,
            ClockEvent::ClockUpdate,
        );
        uninit_field_mut!(*this, clock).write(clock);
    }

    fn realize(&self) -> qemu_api::Result<()> {
        self.char_backend
            .enable_handlers(self, Self::can_receive, Self::receive, Self::event);
        Ok(())
    }
}

```

它们的做的事情基本一致，不同的是 Rust 对 init 的实现，要求更为严格。重点在 init() 的 `ParentInit<Self>` 。

QEMU Rust 为了保证代码是 safe 的，对于 init 阶段，会认为 object 的非 parent 字段，可能是未初始化的。这提醒我们，要在 init 阶段为关键字段初始化，设置 default 值。

一个关键特点，访问 object 的字段，需要使用 `uninit_field_mut!` 。

## read/write 的实现

实现外设的寄存器功能，基本就不依赖 C 的接口了，可以使用 Rust 进行编程，除非涉及到调用 qemu 的一些功能，但是这部分会逐渐被 Rust 封装起来，提供安全的接口。

我们来看一下 C 的实现：

```c
static uint64_t pl011_read(void *opaque, hwaddr offset,
                           unsigned size)
{
    PL011State *s = (PL011State *)opaque;
    uint64_t r;

    switch (offset >> 2) {
    case 0: /* UARTDR */
        r = pl011_read_rxdata(s);
        break;
    case 1: /* UARTRSR */
        r = s->rsr;
        break;
    case 6: /* UARTFR */
        r = s->flags;
        break;
    case 8: /* UARTILPR */
        r = s->ilpr;
        break;
    case 9: /* UARTIBRD */
        r = s->ibrd;
        break;
    case 10: /* UARTFBRD */
        r = s->fbrd;
        break;
    case 11: /* UARTLCR_H */
        r = s->lcr;
        break;
    case 12: /* UARTCR */
        r = s->cr;
        break;
    case 13: /* UARTIFLS */
        r = s->ifl;
        break;
    case 14: /* UARTIMSC */
        r = s->int_enabled;
        break;
    case 15: /* UARTRIS */
        r = s->int_level;
        break;
    case 16: /* UARTMIS */
        r = s->int_level & s->int_enabled;
        break;
    case 18: /* UARTDMACR */
        r = s->dmacr;
        break;
    case 0x3f8 ... 0x400:
        r = s->id[(offset - 0xfe0) >> 2];
        break;
    default:
        qemu_log_mask(LOG_GUEST_ERROR,
                      "pl011_read: Bad offset 0x%x\n", (int)offset);
        r = 0;
        break;
    }

    trace_pl011_read(offset, r, pl011_regname(offset));
    return r;
}
```

再来看看 Rust 的实现：

```rust
    pub(self) fn read(&mut self, offset: RegisterOffset) -> (bool, u32) {
        use RegisterOffset::*;

        let mut update = false;
        let result = match offset {
            DR => self.read_data_register(&mut update),
            RSR => u32::from(self.receive_status_error_clear),
            FR => u32::from(self.flags),
            FBRD => self.fbrd,
            ILPR => self.ilpr,
            IBRD => self.ibrd,
            LCR_H => u32::from(self.line_control),
            CR => u32::from(self.control),
            FLS => self.ifl,
            IMSC => u32::from(self.int_enabled),
            RIS => u32::from(self.int_level),
            MIS => u32::from(self.int_level & self.int_enabled),
            ICR => {
                // "The UARTICR Register is the interrupt clear register and is write-only"
                // Source: ARM DDI 0183G 3.3.13 Interrupt Clear Register, UARTICR
                0
            }
            DMACR => self.dmacr,
        };
        (update, result)
    }
```

可以看到，基本逻辑是一致的。

