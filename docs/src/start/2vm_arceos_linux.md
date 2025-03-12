# ArceOS Linux 2虚拟机启动

1. 编译 linux 镜像，获取 `Image` 镜像文件。

2. 编译 `ArceOS` 的 `Helloworld` 用例，得到二进制文件。

3. 准备虚拟机配置文件

    ```bash
    cd arceos-vmm
    cp configs/vms/arceos-aarch64.toml tmp/
    cp configs/vms/linux-qemu-aarch64-vm2.toml tmp/

    # 编译设备树
    dtc -I dts -O dtb -o tmp/linux-qemu.dtb configs/vms/linux-qemu.dts
    ```

    修改 `tmp/linux-qemu-aarch64-vm2.toml`、`tmp/arceos-aarch64.toml` 文件，将 `kernel_path` 和 `dtb_path` 修改相应的绝对路径。

4. 运行

    ```bash
    make ARCH=aarch64 VM_CONFIGS=tmp/arceos-aarch64.toml:tmp/linux-qemu-aarch64.toml LOG=info BUS=mmio NET=y FEATURES=page-alloc-64g MEM=8g SECOND_SERIAL=y SMP=2 run

    # 之后修改.axconfig.toml，将smp=1 改为smp=2，再重新运行
    ```