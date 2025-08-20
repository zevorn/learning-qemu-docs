TCG 测试框架，主要用来测试 linux-user 模式和 qemu-system-arch 模式下不同体系结构的客户机程序。一般以验证指令集正确性为主。

TCG 测例的特点是，需要为不同 Arch 编写专门的客户机测例，并且在测试的时候，这些测例的源码要进行实时的编译，因此在配置 QEMU 的编译构建选项时，需要加上 Arch 的交叉编译工具链的 prefix（需要自行安装）:

```bash
$(configure) --cross-cc-aarch64=aarch64-cc

 # or

$(configure) --cross-prefix-aarch64=aarch64-linux-elf-
```

编译某个 Arch 的 TCG 测例：

```bash
make build-tcg-tests-$TARGET
```

运行某个 Arch 的 TCG 测例：

```bash
make run-tcg-tests-$TARGET
```

如果想看到更详细的输出，可以在 make 后面添加后缀 `-V`


