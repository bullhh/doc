* [axvcpu](https://github.com/arceos-hypervisor/axvcpu): provides CPU virtualization support
    * highly architecture-dependent
    * stores exception context frame of different architecture
    * basic scheduling item
    * arch-specific vcpu implementations need to be separated into separate crates:
        * [arm_vcpu](https://github.com/arceos-hypervisor/arm_vcpu)
        * [x86_vcpu](https://github.com/arceos-hypervisor/x86_vcpu)
        * [riscv_vcpu](https://github.com/arceos-hypervisor/riscv_vcpu)
        * ...