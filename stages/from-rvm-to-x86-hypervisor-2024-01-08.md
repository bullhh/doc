# 从 RVM 到 x86 Hypervisor（2024/01/08）

本文主要介绍至今（2024/01/08）为止 x86 Hypervisor 方面的最新进展和当前问题。

## 和 RVM 的重要不同

x86 Hypervisor 最初基于 RVM-Tutorial 改动而来，但经过一段时间的开发，已经和 RVM-Tutorial 积累了很多的不同。这里列出比较重要的部分。

### 控制流的改动

在 RVM-Tutorial 中，VC pu 运行的控制流大致如下：

> Hypervisor ➡️ `VmxVcpu::run` ➡️ `VmxVcpu::vmx_launch` ➡️ Guest OS ↔️ `VmxVcpu::vmx_exit` ↔️ `H::vmexit_handler`

Hypervisor 调用 `VmxVcpu::run`，这个函数是永不返回的。`VmxVcpu::run` 通过 `VmxVcpu::vmx_launch` 启动 Guest OS。当 VM-exit 发生时，Cpu 从 Guest OS 中退出到 `VmxVcpu::vmx_exit`，这个函数会调用 `H::vmexit_handler` 处理 VM-exit 然后返回到 Guest OS 中。

这个控制流有几个问题：
1. **不利于切换 VCpu**：如果需要多个 VCpu 在同一个物理 Cpu 上执行，那么唯一可行的调度时机就是在处理 VM-exit 时。然而在这个控制流下，VM-exit Handler 是由当前执行的 VCpu 结构体的成员函数调用的，返回时也必然返回到该处；如果要切换 VCpu，要么需要使用汇编在 VM-exit handler 中手动修改 Rust 的调用栈，要么需要在 VCpu 结构体中实现将控制流转移到其它 VCpu 的逻辑；无论哪种实现方式都比较复杂，且可读性不佳。
2. **不利于实现多 VM 和多 VCpu**：在这个控制流下，VM-exit handler 是通过 `H: HyperCraftHal` 类型参数注入的。因此 `H::vmexit_handler` 中混杂了各类不同层级不同类型的 VM-exit 处理代码，耦合高，不利于扩展到多 VM 和多 VCpu 场景。

为了解决这些问题，x86 Hypervisor修改了这部分代码，引入了 VM 结构体，翻转了 VM-exit handler 部分的控制流。

> Hypervisor ↔️ `VM::run_vcpu` ↔️ `VmxVcpu::run` ↔️ Guest OS

现在遇到 VM-exit 时，Cpu 会从 Guest OS 中退出到 `VmxVcpu::run` 中。`VmxVcpu::run` 会先处理一部分可以由 VCpu 自己处理的 VM-exit，如果不能处理，则交给 VM。

### 虚拟设备的改动

现在虚拟设备分为两批，分别是每 Vcpu 一份的 `PerCpuDevices` 和每 VM 一份的 `PerVMDevices`。同时增加了 MSR 访问相关的接口。

为了适配 Linux，加入或修改了以下虚拟设备的实现：
- 8259 PIC 的实现更完整了；
- 增加了 0x80 调试端口的支持；
- 增加了 8254 PIT 的初步支持；
- PCI 配置端口的初步支持（仅告知 Guest OS 不支持 PCI）

### XSAVE 和浮点相关

XSave 是一套和浮点计算等 Cpu 特性相关的指令集，XSave 的行为由扩展控制寄存器 xcr0 和 MSR IA32_XSS 控制。在切换 Host/Guest 状态时，也要对应切换这两个值。`VmxVcpu` 提供了 `load_guest_xstate` 和 `load_host_xstate` 两个函数负责切换。

同时，CPUID.0DH 的结果也取决于这两个值，因此在执行 CPUID 时也要注意是否需要切换。

## 对 Guest Linux 的支持

### Bootloader

在 x86/x64 平台上的现代的 Linux 内核是不能直接启动的，必须要通过 Bootloader 加载。`apps/hv/guest/vlbl` 目录中提供了一个极简的 Bootloader，并提供了一些最基本的 BIOS 功能。

### Initramfs

要启动 Linux，一个文件系统是必要的。Virtio 是一个通用性很高的选择，但如果要应用到实际的硬件上，则必须让 arceos 实现完整的 Virtio 后端，并且实现磁盘驱动；而在 Qemu 中，虽然可以将 Virtio 磁盘直通给 Guest Linux，但需要实现正确的中断处理。

相比之下，initramfs 可能是个更好的选择。只需要将 initramfs 镜像复制到 Guest 内存空间中，并且让 Bootloader 将其地址和大小提供给 Linux 内核即可。

## 现存的问题

- 关于虚拟设备：
    - `PerCpuDevices` 和 `PerVMDevices` 并未良好区分。很多本应是每 VM 一份的虚拟设备被临时放到了 `PerCpuDevices` 中；
    - `PerCpuDevices` 和 `PerVMDevices` 除了虚拟设备外还要处理其他一些 VM-exit，可能需要更好的命名，或者需要将这一部分拆分出来；
    - 8259 PIC 的实现仍然不够完整；目前仍然不能正确处理 Linux 内核发送的所有指令，但暂时不影响运行；
        - 然而要正确支持多核 VM，正确的中断处理实现仍然是必须的。
    - 8254 PIT 的实现非常简略；定时中断的功能目前实际是硬编码的；
    - 缺少对 PCI/PCI-E 设备的直通和虚拟化支持；
    - 8250/16550 串口的实现不完整，Linux 内核无法识别出其型号，导致 Boot 完成后无法正常作为标准输入输出使用；目前通过直通串口规避此问题。
