前面我们讲了如何使用 qemu 的 helper 函数来模拟指令的功能，但是一般情况下，helper 主要用于 IR 实现不太方便的情况。

如果想要获得更好的性能，推荐使用 IR 来实现。

TCG 的前端负责将目标架构的指令转换为 TCG op，而 TCG 的后端则负责将 TCG op 转换为目标架构的指令。

本节我们主要讲 TCG 的前端，讨论常用的 TCG op 的用法。

!!! note
    推荐阅读：[Documentation/TCG/frontend-ops][1]

TCG op 的基本格式如下：

```
tcg_gen_<op>[i]_<reg_size>(TCGv<reg_size> args, ...)

op: 操作类型
i: 操作数数量
reg_size: 寄存器大小 (32/64/tl)
args: 操作数列表
```

## Registers

```
TCGv reg = tcg_global_mem_new(TCG_AREG0, offsetof(CPUState, reg), "reg");
```

## Temporaries 

```c
// Create a new temporary register
TCGv tmp = tcg_temp_new();

// Create a local temporary register.
// Simple temporary register cannot carry its value across jump/brcond,
// only local temporary can.
TCGv tmpl = tcg_temp_local_new();

// Free a temporary register
tcg_temp_free(tmp);
```

## labels

```c
// Create a new label
int l = gen_new_label();

// Label the current location.
gen_set_label(l);
```

## Ops

操作单个寄存器：

```c
// ret = arg1
// Assignment_(mathematical_logic): Assign one register to another
tcg_gen_mov_tl(ret, arg1);

// ret = - arg1
// Negation: Negate the sign of a register
tcg_gen_neg_tl(ret, arg1);
```

操作两个寄存器：

```c
// ret = arg1 + arg2
// Addition: Add two registers
tcg_gen_add_tl(ret, arg1, arg2);

// ret = arg1 - arg2
// Subtraction: Subtract two registers
tcg_gen_sub_tl(ret, arg1, arg2);

// ret = arg1 * arg2
// Multiplication: Multiply two signed registers and return the result
tcg_gen_mul_tl(ret, arg1, arg2);

// ret = arg1 * arg2
// Multiplication: Multiply two unsigned registers and return the result
tcg_gen_mulu_tl(ret, arg1, arg2);

// ret = arg1 / arg2
// Division_(mathematics): Divide two signed registers and return the result
tcg_gen_div_tl(ret, arg1, arg2);

// ret = arg1 / arg2
// Division_(mathematics): Divide two unsigned registers and return the result
tcg_gen_divu_tl(ret, arg1, arg2);

// ret = arg1 % arg2
// Division_(mathematics): Divide two signed registers and return the remainder
tcg_gen_rem_tl(ret, arg1, arg2);

// ret = arg1 % arg2
// Division_(mathematics) Divide two unsigned registers and return the remainder
tcg_gen_remu_tl(ret, arg1, arg2);
```

## Bit Operations

Logic operations on a single register:

```c
// ret = !arg1
// Negation: Logical NOT an register
tcg_gen_not_tl(ret, arg1);
```

Logic operations on two registers:

```c
// ret = arg1 & arg2
// Logical_conjunction: Logical AND two registers
tcg_gen_and_tl(ret, arg1, arg2);

// ret = arg1 | arg2
// Logical_disjunction: Logical OR two registers
tcg_gen_or_tl(ret, arg1, arg2);

// ret = arg1 ^ arg2
// Exclusive_or: Logical XOR two registers
tcg_gen_xor_tl(ret, arg1, arg2);

// ret = arg1 ↑ arg2
// Logical_NAND: Logical NAND two registers
tcg_gen_nand_tl(ret, arg1, arg2);

// ret = arg1 ↓ arg2
// Logical_NOR Logical NOR two registers
tcg_gen_nor_tl(ret, arg1, arg2);

// ret = !(arg1 ^ arg2)
// Logical_equivalence: Compute logical equivalent of two registers
tcg_gen_eqv_tl(ret, arg1, arg2);

// ret = arg1 & ~arg2
// Logical AND one register with the complement of another
tcg_gen_andc_tl(ret, arg1, arg2);

// ret = arg1 ~arg2
// Logical OR one register with the complement of another
tcg_gen_orc_tl(ret, arg1, arg2);
```

## Shift

```c
// ret = arg1 >> arg2 /* Sign fills vacant bits */
// Arithmetic shift right one operand by magnitude of another
tcg_gen_sar_tl(ret, arg1, arg2);

// ret = arg1 << arg2
// Logical_shift Logical shift left one registerby magnitude of another
tcg_gen_shl_tl(ret, arg1, arg2);

// ret = arg1 >> arg2
// Logical_shift Logical shift right one register by magnitude of another
tcg_gen_shr_tl(ret, arg1, arg2);
```

## Rotation

```c
// ret = arg1 rotl arg2
// Circular_shift: Rotate left one register by magnitude of another
tcg_gen_rotl_tl(ret, arg1, arg2);

// ret = arg1 rotr arg2
// Circular_shift Rotate right one register by magnitude of another
tcg_gen_rotr_tl(ret, arg1, arg2);
```

## Byte

```c
// ret = ((arg1 & 0xff00) >> 8) // ((arg1 & 0xff) << 8)
// Endianness Byte swap a 16bit register
tcg_gen_bswap16_tl(ret, arg1);

// ret = ...see bswap16 and extend to 32bits...
// Endianness Byte swap a 32bit register
tcg_gen_bswap32_tl(ret, arg1);


// ret = ...see bswap32 and extend to 64bits...
// Endianness Byte swap a 64bit register
tcg_gen_bswap64_tl(ret, arg1);

// ret = (int8_t)arg1
// Sign extend an 8bit register
tcg_gen_ext8s_tl(ret, arg1);

// ret = (uint8_t)arg1
// Zero extend an 8bit register
tcg_gen_ext8u_tl(ret, arg1);

// ret = (int16_t)arg1
// Sign extend an 16bit register
tcg_gen_ext16s_tl(ret, arg1);

// ret = (uint16_t)arg1
// Zero extend an 16bit register
tcg_gen_ext16u_tl(ret, arg1);

// ret = (int32_t)arg1
// Sign extend an 32bit register
tcg_gen_ext32s_tl(ret, arg1);

// ret = (uint32_t)arg1
// Zero extend an 32bit register
tcg_gen_ext32u_tl(ret, arg1);

```

## Load/Store

These are for moving data between registers and arbitrary host memory.

Typically used for funky CPU state that is not represented by dedicated
registers already and thus infrequently used.

These are not for accessing the target's memory space;

see the QEMU_XX helpers below for that.

```c
// Load an 8bit quantity from host memory and sign extend
tcg_gen_ld8s_tl(reg, cpu_env, offsetof(CPUState, reg));

// Load an 8bit quantity from host memory and zero extend
tcg_gen_ld8u_tl(reg, cpu_env, offsetof(CPUState, reg));

// Load a 16bit quantity from host memory and sign extend
tcg_gen_ld16s_tl(reg, cpu_env, offsetof(CPUState, reg));

// Load a 16bit quantity from host memory and zero extend
tcg_gen_ld16u_tl(reg, cpu_env, offsetof(CPUState, reg));

// Load a 32bit quantity from host memory and sign extend
tcg_gen_ld32s_tl(reg, cpu_env, offsetof(CPUState, reg));

// Load a 32bit quantity from host memory and zero extend
tcg_gen_ld32u_tl(reg, cpu_env, offsetof(CPUState, reg));

// Load a 64bit quantity from host memory
tcg_gen_ld64_tl(reg, cpu_env, offsetof(CPUState, reg));

// Alias to target native sized load
tcg_gen_ld_tl(reg, cpu_env, offsetof(CPUState, reg));

// Store a 8bit quantity to host memory
tcg_gen_st8_tl(reg, cpu_env, offsetof(CPUState, reg));

// Store a 16bit quantity to host memory
tcg_gen_st16_tl(reg, cpu_env, offsetof(CPUState, reg));

// Store a 32bit quantity to host memory
tcg_gen_st32_tl(reg, cpu_env, offsetof(CPUState, reg));

// Alias to target native sized store
tcg_gen_st_tl(reg, cpu_env, offsetof(CPUState, reg));

```

These are for moving data between registers and arbitrary target memory.

The address to load/store via is always the second argument while the first
argument is always the value to be loaded/stored.

The third argument (memory index) only makes sense for system targets;
user targets will simply specify 0 all the time.

```c
// ret = *(int8_t *)addr
// Load an 8bit quantity from target memory and sign extend
tcg_gen_qemu_ld8s(ret, addr, mem_idx);

// ret = *(uint8_t *)addr
// Load an 8bit quantity from target memory and zero extend
tcg_gen_qemu_ld8u(ret, addr, mem_idx);

// ret = *(int8_t *)addr
// Load a 16bit quantity from target memory and sign extend
tcg_gen_qemu_ld16s(ret, addr, mem_idx);

// ret = *(uint8_t *)addr
// Load a 16bit quantity from target memory and zero extend
tcg_gen_qemu_ld16u(ret, addr, mem_idx);

// ret = *(int8_t *)addr
// Load a 32bit quantity from target memory and sign extend
tcg_gen_qemu_ld32s(ret, addr, mem_idx);

// ret = *(uint8_t *)addr
// Load a 32bit quantity from target memory and zero extend
tcg_gen_qemu_ld32u(ret, addr, mem_idx);

// ret = *(uint64_t *)addr
// Load a 64bit quantity from target memory
tcg_gen_qemu_ld64(ret, addr, mem_idx);

// *(uint8_t *)addr = arg
// Store an 8bit quantity to target memory
tcg_gen_qemu_st8(arg, addr, mem_idx);

// *(uint16_t *)addr = arg
// Store a 16bit quantity to target memory
tcg_gen_qemu_st16(arg, addr, mem_idx);

// *(uint32_t *)addr = arg
// Store a 32bit quantity to target memory
tcg_gen_qemu_st32(arg, addr, mem_idx);

// *(uint64_t *)addr = arg
// Store a 64bit quantity to target memory
tcg_gen_qemu_st64(arg, addr, mem_idx);
```

## Code Flow

```c
// if (arg1 <condition> arg2) goto label
// Test two operands and conditionally branch to a label
tcg_gen_brcond_tl(TCG_COND_XXX, arg1, arg2, label);

// Goto translation block (TB chaining)
// Every TB can goto_tb to max two other different destinations. There are
// two jump slots. tcg_gen_goto_tb takes a jump slot index as an arg,
// 0 or 1. These jumps will only take place if the TB's get chained,
// you need to tcg_gen_exit_tb with (tb // index) for that to ever happen.
// tcg_gen_goto_tb may be issued at most once with each slot index per TB.
tcg_gen_goto_tb(num);

// Exit translation block
// num may be 0 or TB address ORed with the index of the taken jump slot.
// If you tcg_gen_exit_tb(0), chaining will not happen and a new TB
// will be looked up based on the CPU state.
tcg_gen_exit_tb(num);

// ret = arg1 <condition> arg2
// Compare two operands
tcg_gen_setcond_tl(TCG_COND_XXX, ret, arg1, arg2);

```

## Example

我们使用 IR 来实现 cube 指令：

```c
static bool trans_cube(DisasContext *ctx, arg_cube *a)
{
    TCGv dest = tcg_temp_new(); // 申请一个临时变量
    TCGv rd = get_gpr(ctx, a->rd, EXT_NONE); // 获取 rd 寄存器
    // 读取 rs1 寄存器的值指向的内存的值，存储到 dest 中
    tcg_gen_qemu_ld_tl(dest, get_gpr(ctx, a->rs1, EXT_NONE), ctx->mem_idx, MO_TEUQ);
    // 计算 cube 并存储到 rd 寄存器中
    tcg_gen_mul_tl(rd, dest, dest); // rd = dest * dest
    tcg_gen_mul_tl(rd, rd, dest); // rd = rd * dest
    gen_set_gpr(ctx, a->rd, rd);
    return true;
}
```

[1]: https://wiki.qemu.org/Documentation/TCG/frontend-ops