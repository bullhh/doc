# 虚拟中断控制器

## 各架构控制器

* x86_64: local Apic (xAPIC and x2APIC) and IO Apic
* aarch64: GIC (v2,v3,v4)
* riscv64: PLIC or APLIC

## 实现方式

### 虚拟控制器

为相应架构的中断控制器设计虚拟控制器模型，为每个 `vcpu` 创建一个内部中断模拟设备，用于管理 `systime` 中断等，为每个客户机创建一个外部中断模拟设备，用于管理 `io` 中断。

### MMIO 区域注册

在客户机进行中断控制器的寄存器读写时，会通过 `data abort` 陷入到虚拟机，到达 `axvcpu` 的如下代码段：

```rust
match &exit_reason {
    AxVCpuExitReason::MmioRead {
        addr,
        width,
        reg,
        reg_width: _,
    } => {
        let val = self
            .get_devices()
            .handle_mmio_read(*addr, (*width).into())?;
        vcpu.set_gpr(*reg, val);
        true
    }
    AxVCpuExitReason::MmioWrite { addr, width, data } => {
        self.get_devices()
            .handle_mmio_write(*addr, (*width).into(), *data as usize);
        true
    }
    AxVCpuExitReason::IoRead { port: _, width: _ } => true,
    AxVCpuExitReason::IoWrite {
        port: _,
        width: _,
        data: _,
    } => true,
    AxVCpuExitReason::NestedPageFault { addr, access_flags } => self
        .inner_mut
        .address_space
        .lock()
        .handle_page_fault(*addr, *access_flags),
    _ => false,
};
```

通过 `handle_mmio_read` 和 `handle_mmio_write` 实现相应 `mmio` 范围内的地址访问会路由到相应虚拟设备。

通过对相应虚拟寄存器的读写逻辑，实现对客户机中断设置的权限控制。

### 中断透传

虚拟中断控制器通过配置表，判断客户机是否有权限控制中断号，若有权限，则将客户机中断号相应操作透传到物理中断控制器。

### 虚拟中断

系统时钟和虚拟设备等中断通过全虚拟化方式实现，每个 `vcpu` 都有一个中断向量表，用于记录客户机中断号对应的中断状态。

当虚拟设备触发中断时，向物理中断控制器发送软中断，由物理中断控制器将中断请求转发到 `vcpu`。

## AARCH64 & RISCV64

中断控制器作为虚拟设备，位于 `Axvm` 层，实现通用接口，供 `vcpu` 和上层使用。

## X86_64

TODO

## 当前状态

目前通过直通方式绕过虚拟中断控制器，直接使用物理中断控制器。

## 参考资料

* Intel® 64 and IA-32 Architectures Software Developer’s Manual, Volum 3C: CHAPTER 30 APIC VIRTUALIZATION AND VIRTUAL INTERRUPTS
* [ARM Generic Interrupt Controller Architecture version 2.0 - Architecture Specification](https://developer.arm.com/documentation/ihi0048/latest/)
* [Arm Generic Interrupt Controller (GIC) Architecture Specification GIC architecture version 3 and version 4](https://developer.arm.com/documentation/ihi0069/latest/)
* https://github.com/riscv/riscv-plic-spec/blob/master/riscv-plic.adoc