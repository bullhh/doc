# 虚拟中断控制器

## 各架构控制器

* x86_64: local Apic (xAPIC and x2APIC) and IO Apic
* aarch64: GIC (v2,v3,v4)
* riscv64: PLIC or APLIC

## AARCH64 & RISCV64

中断控制器作为虚拟设备，位于 `Axvm` 层，实现通用接口，供 `vcpu` 和上层使用。

## 当前状态

目前通过直通方式绕过虚拟中断控制器，直接使用物理中断控制器。

## 参考资料

* Intel® 64 and IA-32 Architectures Software Developer’s Manual, Volum 3C: CHAPTER 30 APIC VIRTUALIZATION AND VIRTUAL INTERRUPTS
* [ARM Generic Interrupt Controller Architecture version 2.0 - Architecture Specification](https://developer.arm.com/documentation/ihi0048/latest/)
* [Arm Generic Interrupt Controller (GIC) Architecture Specification GIC architecture version 3 and version 4](https://developer.arm.com/documentation/ihi0069/latest/)
* https://github.com/riscv/riscv-plic-spec/blob/master/riscv-plic.adoc