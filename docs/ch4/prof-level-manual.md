本文档是专业阶段的实验手册。

专业阶段的实验围绕着 G233 虚拟开发板展开，你需要按照 [G233 Board Datasheet][5] 手册给出的硬件参数，完成实验任务，从而帮助你更好的掌握 QEMU 建模，理解 QEMU 模拟器的工作原理。

!!! note "温馨提示"

    关于硬件建模部分，你可以尝试用 Rust 来实现（QEMU 有基本框架）；若想进一步挑战自我，也可以尝试用 Rust 模拟客户机指令（需要自己从零实现）。

## 环境搭建

第一步，以 Ubuntu 22.04 为例，介绍如何安装 QEMU 开发环境。

```bash
# 备份 sources.list 文件
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak

# 启用 deb-src 源（将所有 deb 源对应的 deb-src 源解锁）
sudo sed -i '/^# deb-src /s/^# //' /etc/apt/sources.list
sudo apt-get update
sudo apt update && sudo apt build-dep qemu

# 创建工具链安装目录
sudo mkdir -p /opt/riscv

# 下载工具链压缩包
wget https://github.com/riscv-collab/riscv-gnu-toolchain/releases/download/2025.09.28/riscv64-elf-ubuntu-22.04-gcc-nightly-2025.09.28-nightly.tar.xz -O riscv-toolchain.tar.xz

# 解压到安装目录
sudo tar -xJf riscv-toolchain.tar.xz -C /opt/riscv --strip-components=1

# 设置权限
sudo chown -R $USER:$USER /opt/riscv
echo "/opt/riscv/bin" >> $GITHUB_PATH
export PATH=$PATH:/opt/riscv/bin/

riscv64-unknown-elf-gcc --version  # 验证编译器是否可用

# 安装 Rust 工具链
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y --default-toolchain stable

# 验证 Rust 安装
rustup --version
rustc --version
cargo --version
```

!!! note "提示"

    安装 QEMU 开发环境，请参考导学阶段的 [Step0: 搭建 QEMU 开发环境][1]。

    安装 RISC-V 的交叉编译工具链：[下载地址][2]，尽量选择最新的版本，要求安装 `riscv64-unknown-elf-` 类型。

    安装 Rust，版本要求 >= 1.85，安装方法请参考 [Rust 官方文档][6]。


第二步，点击[这里][3]，自动 fork 作业仓库到 GTOC 组织下面，该仓库会为你开通代码上传权限。

第三步，需要 clone 刚刚 fork 好的仓库到本地：

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
            --cross-prefix-riscv64=riscv64-unknown-elf- \
            --enable-rust
```

执行时，如果看到以下输出，证明交叉编译工具链配置成功：

```bash
  ...
  Cross compilers
    riscv64                         : riscv64-unknown-elf-gcc
  ...
```

## 提交代码

所有实验的测题源码，均放在仓库根目录路径： `tests/gevico/tcg/riscv64/` 。

你需要熟读每个测题源码，理解每个测题的测试意图，并实现对应的 QEMU 建模功能（需要修改 QEMU 本体源码，非测题源码），文末会给出具体实验的介绍，辅助你阅读测题源码。

每次实验完成后，需要将你的代码提交到你的 fork 仓库中。

```bash
git add .
git commit -m "feat: subject..."
git push origin main
```

!!! note

    请确保你的代码符合仓库的代码规范，包括代码格式、注释等。

## 测评验收


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

每道测题 10 分，一共 10 道测题，共计 100 分，评分将显示到训练营的[专业阶段排行榜][4]。

## 实验介绍

为了方便设计测题，我们设计了一个虚拟板卡 G233，并且编写了 [G233 Board Datasheet][5]，用于描述 G233 板卡的硬件规格和功能。

该阶段涉及的所有实验的硬件参数，全部记录在 [G233 Board Datasheet][5] 中。熟读手册可以帮你更好的理解每个实验的测试意图。

所有实验的测题，均在 `tests/gevico/tcg/riscv64/` 目录下，以 `test-` 开头的 `.c` 文件。

### 实验一 test-board-g233

源码路径： `tests/gevico/tcg/riscv64/test-board-g233.c`。

该实验用于验证 G233 Board 是否正常工作。你需要在 QEMU 中模拟 G233 Board。

基本代码已经存放在 `hw/riscv/g233.c` 中，需要你进一步补全它。

### 实验二 test-insn-dma

源码路径： `tests/gevico/tcg/riscv64/test-insn-dma.c`。

该实验用于验证 G233 Board 的 DMA 指令功能是否正常工作。你需要在 QEMU 中实现这条指令。

这条指令的详细描述在 [G233 Board Datasheet][5] 中。

### 实验三 test-insn-sort

源码路径： `tests/gevico/tcg/riscv64/test-insn-sort.c`。

该实验用于验证 G233 Board 的 sort 指令功能是否正常工作。你需要在 QEMU 中实现这条指令。

这条指令的详细描述在 [G233 Board Datasheet][5] 中。

### 实验四 test-insn-crush

源码路径： `tests/gevico/tcg/riscv64/test-insn-crush.c`。

该实验用于验证 G233 Board 的 crush 指令功能是否正常工作。你需要在 QEMU 中实现这条指令。

这条指令的详细描述在 [G233 Board Datasheet][5] 中。

### 实验五 test-insn-expand

源码路径： `tests/gevico/tcg/riscv64/test-insn-expand.c`。

该实验用于验证 G233 Board 的 expand 指令功能是否正常工作。你需要在 QEMU 中实现这条指令。

这条指令的详细描述在 [G233 Board Datasheet][5] 中。

### 实验六 test-spi-jedec

源码路径： `tests/gevico/tcg/riscv64/test-spi-jedec.c`。

该实验用于验证 G233 Board 的 SPI-JEDEC 功能是否正常工作。你需要在 QEMU 中实现这个外设。

这个外设的详细描述在 [G233 Board Datasheet][5] 中。

### 实验七 test-flash-read

源码路径： `tests/gevico/tcg/riscv64/test-flash-read.c`。

该实验用于验证 G233 Board 的 flash-read 功能是否正常工作。你需要在 QEMU 中实现这个外设。

这个外设的详细描述在 [G233 Board Datasheet][5] 中。

### 实验八 test-flash-read-int

源码路径： `tests/gevico/tcg/riscv64/test-flash-read-interrupt.c`。

该实验用于验证 G233 Board 的中断功能是否正常工作。你需要在 QEMU 中实现这个外设的中断功能。

这个中断对应的外设的详细描述在 [G233 Board Datasheet][5] 中。

### 实验九 TODO
[待更新]

### 实验十 TODO
[待更新]

[1]: https://qemu.readthedocs.io/en/v10.0.3/devel/build-environment.html
[2]: https://github.com/riscv-collab/riscv-gnu-toolchain/releases/
[3]: https://classroom.github.com/a/HXuCy8g7
[4]: https://opencamp.cn/qemu/camp/2025/stage/3?tab=rank
[5]: https://gevico.github.io/learning-qemu-docs/ch4/g233-board-datasheet/
[6]: https://rust-lang.org/zh-CN/tools/install/
