本文档是参与专业阶段测评的用户手册。

专业阶段的测评围绕着 G233 虚拟开发板展开，你需要按照 [G233 Board Datasheet][5] 的硬件参数，完成下面实验给出任务，从而帮助你更好的掌握 QEMU 建模，理解 QEMU 模拟器的工作原理。

!!! note "温馨提示"

    关于硬件建模的部分，你可以尝试使用 Rust 实现，如果想要挑战一下自我，可以尝试用 Rust 模拟客户机指令，但这部分不做强制要求。

## 环境搭建

第一步，需要安装 QEMU 开发环境，请参考导学阶段的[Step0: 搭建 QEMU 开发环境][1]。

另外需要安装 RISC-V 的交叉编译工具链：[下载地址][2]，尽量选择最新的版本，要求安装 `riscv64-unknown-elf-` 类型。

!!! note

    如果你想尝试使用 Rust 开发，请自行安装 Rust 工具链，version >= 1.85

第二步，点击[这里][3]，自动 fork 作业仓库到 GTOC 组织下面，该仓库会为你开通代码上传权限，便于上传你的作业。

第三步，需要 clone 刚刚 fork 好的仓库的到本地：

```bash
git clone git@github.com:gevico/learning-qemu-2025-<你的 github 用户名>.git

# 比如 github 用户名是 zevorn，那么命令如下：
# git clone git@github.com:gevico/learning-qemu-2025-zevorn.git
```

第四步，添加上游远程仓库，用于同步上游的代码变更：

```bash
git remote add upstream git@github.com:gevico/gevico-classroom-learning-qemu-2025-learning-qemu.git
```

同步上游代码变更的常用命令：

```bash
git pull upstream main --rebase
```

最后一步，配置编译选项：

```bash
cd qemu
./configure --target-list=riscv64-softmmu \
            --extra-cflags="-O0 -g3" \
            --cross-prefix-riscv64=riscv64-unknown-elf-
```

如果使用 Rust 开发，需要追加编译选项 `--enable-rust`


执行时，如果看到以下输出，证明交叉编译工具链配置成功：

```bash
  ...
  Cross compilers
    riscv64                         : riscv64-unknown-elf-gcc
  ...
```


## 测评验收

所有实验的测题源码，均放在仓库根目录路径： `tests/gevico/tcg/` 。

本地运行测题的方式：

```bash
make check-gevico-tcg
```

全部测题通过的情况下，你会看到如下输出：

```bash
  BUILD   riscv64-softmmu guest-tests
  RUN     riscv64-softmmu guest-tests
  TEST      1/10   test-board-g233 on riscv64
  TEST      2/10   test-insn-dma on riscv64
  TEST      3/10   test-insn-sort on riscv64
  TEST      4/10   test-insn-crush on riscv64
  TEST      5/10   test-insn-expand on riscv64
  ...
```

如果你想运行某个测例，比如 `test-board-g233`，可以使用如下命令：

```bash
make -C build/tests/gevico/tcg/riscv64-softmmu/  run-board-g233
```

!!! note

    当你使用 `make -C` 指定了路径以后，你可以通过输入 `run-` 和 tab 键来查看可以运行的测题

如果你想调试某个测例，比如 `test-board-g233`，可以使用如下命令启用 QEMU 的远程调试功能：

```bash
make -C build/tests/gevico/tcg/riscv64-softmmu gdbstub-board-g233
```

同理，你也可以通过 `gdbstub-` 和 tab 键来查看可以远程调试的测例。

然后需要你本地另起一个终端，使用 riscv-elf-gdb 加载被调试客户机二进制程序，进行远程调试。

!!! note

    你需要熟读 G233 Board Datasheet 和测题的源码，来理解每个实验的测试意图，这会极大地方便你调试，提高开发效率。

评分将显示到训练营的[专业阶段排行榜][4]。


[1]: https://qemu.readthedocs.io/en/v10.0.3/devel/build-environment.html
[2]: https://github.com/riscv-collab/riscv-gnu-toolchain/releases/
[3]: https://classroom.github.com/a/HXuCy8g7
[4]: https://opencamp.cn/qemu/camp/2025/stage/3?tab=rank
[5]: https://gevico.github.io/learning-qemu-docs/ch4/g233-board-datasheet/
