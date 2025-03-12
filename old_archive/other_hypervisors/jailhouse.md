# [Jailhouse](https://github.com/siemens/jailhouse)

## Enable Jailhouse

目前 arceos-vmm 的启动就是部分利用了这个功能。

## Boot Guest Cell

大概包括三步：
    1. create cell
    2. load cell
    3. start cell
    ```bash
        jailhouse cell create /path/to/apic-demo.cell
        jailhouse cell load apic-demo /path/to/apic-demo.bin
        jailhouse cell start apic-demo
    ```

### cell_create

*  `jailhouse_cmd_cell_create`
    * `cell_create`
        * 大概就是创建 cell config
        * `jailhouse_cell_pci_devices`
            * 也是在拷贝 pci_devices 的配置信息
        * `jailhouse_sysfs_cell_create`
            * 在 sysfs 中为该 cell 添加一组属性
            * 以便用户空间程序可以查询和配置这个 cell ？
    * `for_each_cpu(cpu, &cell->cpus_assigned)`
        * `cpu_down` 关掉目标核心
    * `jailhouse_pci_do_all_devices`
        * 传入的第三个参数： action 为 `JAILHOUSE_PCI_ACTION_CLAIM`
        * 遍历这个 cell 拥有的所有 pci 设备
        * `jailhouse_pci_claim_release`
            * `device_release_driver`
                * Linux 内核释放这个 pci 设备所关联的驱动程序
            * `pci_add_dynid`
                * 把这个 pci 设备暂时先挂载到 `jailhouse_pci_stub_driver` 驱动下面，详见这个驱动的[说明](https://github.com/siemens/jailhouse/blob/master/driver/pci.c#L77)
    * `jailhouse_call_arg1(JAILHOUSE_HC_CELL_CREATE, __pa(config));`
        * HVC 调用 hypervisor 的 `cell_create`
            * cell_suspend(&root_cell);
                * 除了当前 cpu 外先暂停其他核心防止 data races
            * 进行 root-mode 里面的 guest cell 创建操作，vcpu之类的
            * **`cell_init`**
                * `mmio_cell_init`
                    *  `for_each_unit(unit)`
                        * pci 设备
                            * `hypervisor/pci.c`
                            * `pci_mmio_count_regions`
                        * x86相关的 unit
                            1. `hypervisor/arch/x86/cat.c`
                                * Cache Allocation Technology
                            2. `hypervisor/arch/x86/ioapic.c`
                                * IOAPIC
                            3. `hypervisor/arch/x86/vtd.c`
                                * VT-d
            * **`arch_cell_create`**
                * `vcpu_cell_init`
                    * `vcpu_vendor_cell_init`
                        * 位于 `hypervisor/arch/x86/vmx.c`
                        * allocate io_bitmap
                        * build root EPT of cell
                        * Map the special APIC access page at the default address (XAPIC_BASE)
                * copy io bitmap from cell config
                * Shrink PIO access of root cell
                * permit access to the PM timer
            * 各个 unit 的 cell_init（pci，ioapic之类的）
                ```C
                  for_each_unit(unit) {
                    err = unit->cell_init(cell);
                  }
                ```
                * `pci_cell_init`
                    * 注册 `pci_mmconfig_access_handler`
                    * 从 root cell 中移除 device
                        ```C
                        root_device = pci_get_assigned_device(&root_cell,
						      dev_infos[ndev].bdf);
                        if (root_device)
                            pci_remove_physical_device(root_device);
                        ```
                    * 将 device 加到这个 guest cell 里面
                        * `pci_add_physical_device`
                            * `arch_pci_add_physical_device`
                                * `iommu_add_pci_device`
                                    * 定义在 `hypervisor/arch/x86/vtd.c`
                                    * 
                            * 建立对 msix_table 的映射
                            * 注册 mmio handler `pci_msix_access_handler`
                            * `pci_reset_device(device);`
                        * 读取 msi/msix 信息并保存
                            ```C
                            for_each_pci_cap(cap, device, ncap)
                                if (cap->id == PCI_CAP_MSI)
                                    pci_save_msi(device, cap);
                                else if (cap->id == PCI_CAP_MSIX)
                                    pci_save_msix(device, cap);
                            ```
            * Shrinking the new cell's CPUs
            * 解除 与 root cell 的内存映射
            * 建立 guest cell 的内存映射
                * `arch_map_memory_region`
                    * `vcpu_map_memory_region`
                        * 填 ept 的页表项
                    * `iommu_map_memory_region`
                        * 填 iommu 的页表项
            * `config_commit(cell);`
                * Apply system configuration changes
                * `arch_flush_cell_vcpu_caches`
                * `arch_config_commit`
                    * `iommu_config_commit`
                        * 位于 `hypervisor/arch/x86/vtd.c`
                    * `ioapic_config_commit`
                * `pci_config_commit`
                    * PCI_CAP_MSI
                        * `arch_pci_update_msi`
                    * PCI_CAP_MSIX
                        * `pci_update_msix`
                        * `pci_suppress_msix`
            * cell_resume(&root_cell);
    * `cell_register(cell)`
        * `list_add_tail(&cell->entry, &cells);`
        * `jailhouse_sysfs_cell_register(cell);`

### cell_load
* `jailhouse_cmd_cell_load`
    * `jailhouse_call_arg1(JAILHOUSE_HC_CELL_SET_LOADABLE, cell->id)`
        * HVC 调用 hypervisor 的 `cell_set_loadable`
            * 首先暂停相关的cpu
            * 调用 `remap_to_root_cell` 重建 loadable memory regions 到 root cell 内存空间的地址映射
            * cell_resume(&root_cell)
    * `load_image(cell, image)`
        * 调用 `jailhouse_ioremap` 重建 host Linux 到 guest physical addr 的映射
            * 传入的 `vaddr` 为 0, 返回 `image_mem` 为 Linux 内核虚拟地址
            ```
                image_mem = jailhouse_ioremap(phys_start, 0,
				      PAGE_ALIGN(image.size + page_offs));
            ```
        * `copy_from_user` 直接拷贝到目标 guest physical addr
            ```
                	if (copy_from_user(image_mem + page_offs,
                            (void __user *)(unsigned long)image.source_address,
                            image.size))
                        err = -EFAULT;
            ```

### cell_start
* `jailhouse_cmd_cell_start`
    * `jailhouse_call_arg1(JAILHOUSE_HC_CELL_START, cell->id)`
        * HVC 调用 hypervisor 的 `cell_start`， 启动目标 cell
            * 调用 `unmap_from_root_cell` 解除 root cell 对这个 cell 的 image 的地址映射
            * 调用 `config_commit(NULL);`
                * Apply system configuration changes.
            * 设置 `comm_region`, a consistent Communication Region state to the cell
            * `pci_cell_reset(cell);`
                    ```C
                    for_each_configured_pci_device(device, cell)
                        if (device->cell)
                            pci_reset_device(device);
                    ```
                * `pci_reset_device`
                    * 有一大堆的 `pci_write_config` 操作
	        * `arch_cell_reset(cell);`
            * `for_each_cpu(cpu, cell->cpu_set)`
                * `arch_reset_cpu(cpu);`
            * `cell_resume(&root_cell);`
                * 恢复所有 cell 的运行，这时新初始化的cell开始运行

## PCI device
### 初始化

### 运行时处理
* 主要入口 `vcpu_handle_exit`
    * 主要包括 pio 与 mmio 陷入的处理
    ```C
    void vcpu_handle_exit(struct per_cpu *cpu_data)
    {
        struct vmcb *vmcb = &cpu_data->vmcb;
        switch (vmcb->exitcode) {
        case VMEXIT_NPF:
            if ((vmcb->exitinfo1 & 0x7) == 0x7 &&
                vmcb->exitinfo2 >= XAPIC_BASE &&
                vmcb->exitinfo2 < XAPIC_BASE + PAGE_SIZE) {
                /* APIC access in non-AVIC mode */
                cpu_public->stats[JAILHOUSE_CPU_STAT_VMEXITS_XAPIC]++;
                if (svm_handle_apic_access(vmcb))
                    goto vmentry;
            } else {
                /* General MMIO (IOAPIC, PCI etc) */
                cpu_public->stats[JAILHOUSE_CPU_STAT_VMEXITS_MMIO]++;
                if (vcpu_handle_mmio_access())
                    goto vmentry;
            }
            break;
        case VMEXIT_IOIO:
            cpu_public->stats[JAILHOUSE_CPU_STAT_VMEXITS_PIO]++;
            if (vcpu_handle_io_access())
                goto vmentry;
            break;
            }
    ```
* mmio
    * `vcpu_handle_mmio_access`
        * `mmio_handle_access`
            * 根据mmio访问的内存范围调用在 `pci_cell_init` 中注册好的 handler
                * `pci_cell_init` 中注册的 `pci_mmconfig_access_handler`
                * `pci_add_physical_device` 中注册的 `pci_msix_access_handler`
* pio
    * `vcpu_handle_io_access`
        * `vcpu_vendor_get_io_intercept(&io);`
            * 首先从 vmcs 中解析 io 拦截信息
        * `x86_pci_config_handler`
            * 模拟 port io 访问

