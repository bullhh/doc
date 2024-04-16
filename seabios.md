# [Seabios](https://github.com/coreboot/seabios/)

考虑使用一个完备的 bios 来代替现在苏佬捏的 vlbl

## Documentation

* [memory model](https://www.seabios.org/Memory_Model)

* [execution and code flow](https://www.seabios.org/Execution_and_code_flow)

* [linking overview](https://www.seabios.org/Linking_overview)

## misc

* bios 入口
    * `reset_vector`
    * as mensioned in [execution and code flow](https://www.seabios.org/Execution_and_code_flow)
        * > On emulators, this phase starts when the CPU starts execution in 16bit
mode at 0xFFFF0000:FFF0. 
        * 入口代码
        ```asm
                ORG 0xfff0 // Power-up Entry Point
                .global reset_vector
        reset_vector:
                ljmpw $SEG_BIOS, $entry_post
        ```
        * 重置后系统状态
            * cs.selector = 0xf000
            * cs.base = 0xffff0000
            * rip = 0xfff0
        * 需要注意包括 0xfffffff0 在内的这部分 IPA 在 gpm 里面的映射 
    * 跳转到 `post.c:handle_post()` in 32bit mode
        ```asm
                ORG 0xe05b
        entry_post:
                cmpl $0, %cs:HaveRunPost                // Check for resume/reboot
                jnz entry_resume
                ENTRY_INTO32 _cfunc32flat_handle_post   // Normal entry point
        ```

* `paravirt.c`
    * QEMU将 VM的启动引导顺序、ACPI 和 SMBIOS表、SMP 和 NUMA 信息等传递给虚拟机。
    * QEMU 的 Firmware Configuration(fw_cfg) Device 机制是 hypervisor 想要支持 seabios 的运行需要完成的

## .config

```
#
# Automatically generated file; DO NOT EDIT.
# SeaBIOS Configuration
#

#
# General Features
#
# CONFIG_COREBOOT is not set
CONFIG_QEMU=y
# CONFIG_CSM is not set
CONFIG_QEMU_HARDWARE=y
# CONFIG_XEN is not set
# CONFIG_THREADS is not set
# CONFIG_RELOCATE_INIT is not set
# CONFIG_BOOTMENU is not set
# CONFIG_BOOTORDER is not set
# CONFIG_HOST_BIOS_GEOMETRY is not set
CONFIG_ENTRY_EXTRASTACK=y
# CONFIG_MALLOC_UPPERMEMORY is not set
CONFIG_ROM_SIZE=0

#
# Hardware support
#
# CONFIG_USB is not set
CONFIG_SERIAL=y
CONFIG_SERCON=y
# CONFIG_LPT is not set
# CONFIG_HARDWARE_IRQ is not set
# CONFIG_USE_SMM is not set
# CONFIG_MTRR_INIT is not set
# CONFIG_PMTIMER is not set
# CONFIG_TSC_TIMER is not set

#
# BIOS interfaces
#
# CONFIG_DRIVES is not set
# CONFIG_PCIBIOS is not set
# CONFIG_APMBIOS is not set
# CONFIG_PNPBIOS is not set
CONFIG_OPTIONROMS=y
# CONFIG_PMM is not set
CONFIG_BOOT=y
# CONFIG_KEYBOARD is not set
# CONFIG_MOUSE is not set
# CONFIG_S3_RESUME is not set
# CONFIG_VGAHOOKS is not set
# CONFIG_DISABLE_A20 is not set
# CONFIG_WRITABLE_UPPERMEMORY is not set

#
# BIOS Tables
#
# CONFIG_PIRTABLE is not set
# CONFIG_MPTABLE is not set
# CONFIG_SMBIOS is not set
# CONFIG_ACPI is not set
# CONFIG_FW_ROMFILE_LOAD is not set
# CONFIG_ACPI_PARSE is not set

#
# VGA ROM
#
CONFIG_NO_VGABIOS=y
# CONFIG_VGA_STANDARD_VGA is not set
# CONFIG_VGA_CIRRUS is not set
# CONFIG_VGA_ATI is not set
# CONFIG_VGA_BOCHS is not set
# CONFIG_VGA_GEODEGX2 is not set
# CONFIG_VGA_GEODELX is not set
# CONFIG_DISPLAY_BOCHS is not set
# CONFIG_VGA_RAMFB is not set
# CONFIG_BUILD_VGABIOS is not set
CONFIG_VGA_EXTRA_STACK_SIZE=512

#
# Debugging
#
CONFIG_DEBUG_LEVEL=1
CONFIG_DEBUG_SERIAL=y
CONFIG_DEBUG_SERIAL_PORT=0x3f8
CONFIG_DEBUG_IO=y

```