Provided by @[TSM](https://github.com/915604903T) 

- fn main() 
  - fn run()
    - let mut vm_config: VmConfig = create_vmconfig(&cmd_args)?; 创建vmconfig，和qemu一样的语句格式
      - 
    - real_main(cmd_args: &arg_parser::ArgMatches, vm_config: &mut VmConfig)
       - StdMachine::new(vm_config).with_context(|| "Failed to init StandardVM")? 
         - add_args_to_config_multi!((args.values_of("drive")), vm_cfg, add_drive);
           - add_drive
             - add_block_drive(对于virtio而言)
         - add_args_to_config_multi!((args.values_of("device")), vm_cfg, add_device);
           - add_device 把类型virtio-blk-pci提取，后面按照string存
       - MachineOps::realize(&vm, vm_config).with_context(|| "Failed to realize standard VM.")?;
         - locked_vm.add_devices(vm_config)?;
           - create_device_add_matches! 
             - add_virtio_pci_blk
               - add_virtio_pci_device(&device_cfg.id, &bdf, device.clone(), multi_func, false) （need_irqfd为false）
                 - VirtioPciDevice::new
                 - pcidev.realize()  初始化pci config space的一些配置
                 

```rust
pub fn new(
        name: String,
        devfn: u8,
        sys_mem: Arc<AddressSpace>,
        device: Arc<Mutex<dyn VirtioDevice>>,
        parent_bus: Weak<Mutex<PciBus>>,
        multi_func: bool,
    ) -> Self {
        let queue_num = device.lock().unwrap().queue_num();
        VirtioPciDevice {
            base: PciDevBase {
                base: DeviceBase::new(name, true),
                config: PciConfig::new(PCIE_CONFIG_SPACE_SIZE, VIRTIO_PCI_BAR_MAX),
                devfn,
                parent_bus,
            },
            device,
            dev_id: Arc::new(AtomicU16::new(0)),
            sys_mem,
            cfg_cap_offset: 0,
            notify_eventfds: Arc::new(NotifyEventFds::new(queue_num)),	
            interrupt_cb: None,
            multi_func,
            need_irqfd: false, // 用户态处理中断才需要irqfd？
        }
    }
```

# impl **PciDevOps** for **VirtioPciDevice**

```rust
fn realize(mut self) -> Result<()> {
        self.init_write_mask(false)?;
        self.init_write_clear_mask(false)?;

        let device_quirk = self.device.lock().unwrap().device_quirk();
        let device_type = self.device.lock().unwrap().device_type();
        le_write_u16(
            &mut self.base.config.config,
            VENDOR_ID as usize,
            VIRTIO_PCI_VENDOR_ID,
        )?;
        le_write_u16(
            &mut self.base.config.config,
            DEVICE_ID as usize,
            VIRTIO_PCI_DEVICE_ID_BASE + device_type as u16,
        )?;
        self.base.config.config[REVISION_ID] = VIRTIO_PCI_ABI_VERSION;
        let class_id = get_virtio_class_id(device_type, device_quirk);
        le_write_u16(
            &mut self.base.config.config,
            SUB_CLASS_CODE as usize,
            class_id,
        )?;
        le_write_u16(
            &mut self.base.config.config,
            SUBSYSTEM_VENDOR_ID,
            VIRTIO_PCI_VENDOR_ID,
        )?;
        // For compatibility with windows viogpu as front-end drivers.
        let subsysid = if device_type == VIRTIO_TYPE_GPU {
            PCI_SUBDEVICE_ID_QEMU
        } else {
            0x40 + device_type as u16
        };
        le_write_u16(&mut self.base.config.config, SUBSYSTEM_ID, subsysid)?;

        init_multifunction(
            self.multi_func,
            &mut self.base.config.config,
            self.base.devfn,
            self.base.parent_bus.clone(),
        )?;
        #[cfg(target_arch = "aarch64")]
        self.base.config.set_interrupt_pin();
		// 下面msix之前的cap都是在初始化vendor自定义的cap
        let common_cap = VirtioPciCap::new(
            size_of::<VirtioPciCap>() as u8 + PCI_CAP_VNDR_AND_NEXT_SIZE,
            VirtioPciCapType::Common as u8,
            VIRTIO_PCI_MEM_BAR_IDX,
            VIRTIO_PCI_CAP_COMMON_OFFSET,
            VIRTIO_PCI_CAP_COMMON_LENGTH,
        );
        self.modern_mem_region_map(common_cap)?;
	/* 具体写cap config的信息
	fn modern_mem_region_map<T: ByteCode>(&mut self, data: T) -> Result<usize> {
        let cap_offset = self.base.config.add_pci_cap(
            PCI_CAP_ID_VNDR,	// 9
            size_of::<T>() + PCI_CAP_VNDR_AND_NEXT_SIZE as usize,
        )?;

        let write_start = cap_offset + PCI_CAP_VNDR_AND_NEXT_SIZE as usize;
        self.base.config.config[write_start..(write_start + size_of::<T>())]
            .copy_from_slice(data.as_bytes());

        Ok(write_start)
    }
    /// Add a pci standard capability in the configuration space.
    ///
    /// # Arguments
    ///
    /// * `id` - Capability ID.
    /// * `size` - Size of the capability.
    是pci_config实现的函数
    pub fn add_pci_cap(&mut self, id: u8, size: usize) -> Result<usize> {
        let offset = self.last_cap_end as usize;
        if offset + size > PCI_CONFIG_SPACE_SIZE {
            return Err(anyhow!(PciError::AddPciCap(id, size)));
        }

        self.config[offset] = id;
        self.config[offset + NEXT_CAP_OFFSET as usize] = self.config[CAP_LIST as usize];
        self.config[CAP_LIST as usize] = offset as u8;
        self.config[STATUS as usize] |= STATUS_CAP_LIST as u8;

        let regs_num = if size % REG_SIZE == 0 {
            size / REG_SIZE
        } else {
            size / REG_SIZE + 1
        };
        for _ in 0..regs_num {
            le_write_u32(&mut self.write_mask, self.last_cap_end as usize, 0)?;
            self.last_cap_end += REG_SIZE as u16;
        }

        Ok(offset)
    }*/
        let isr_cap = VirtioPciCap::new(
            size_of::<VirtioPciCap>() as u8 + PCI_CAP_VNDR_AND_NEXT_SIZE,
            VirtioPciCapType::ISR as u8,
            VIRTIO_PCI_MEM_BAR_IDX,
            VIRTIO_PCI_CAP_ISR_OFFSET,
            VIRTIO_PCI_CAP_ISR_LENGTH,
        );
        self.modern_mem_region_map(isr_cap)?;

        let device_cap = VirtioPciCap::new(
            size_of::<VirtioPciCap>() as u8 + PCI_CAP_VNDR_AND_NEXT_SIZE,
            VirtioPciCapType::Device as u8,
            VIRTIO_PCI_MEM_BAR_IDX,
            VIRTIO_PCI_CAP_DEVICE_OFFSET,
            VIRTIO_PCI_CAP_DEVICE_LENGTH,
        );
        self.modern_mem_region_map(device_cap)?;

        let notify_cap = VirtioPciNotifyCap::new(
            size_of::<VirtioPciNotifyCap>() as u8 + PCI_CAP_VNDR_AND_NEXT_SIZE,
            VirtioPciCapType::Notify as u8,
            VIRTIO_PCI_MEM_BAR_IDX,
            VIRTIO_PCI_CAP_NOTIFY_OFFSET,
            VIRTIO_PCI_CAP_NOTIFY_LENGTH,
            VIRTIO_PCI_CAP_NOTIFY_OFF_MULTIPLIER,
        );
        self.modern_mem_region_map(notify_cap)?;

        let cfg_cap = VirtioPciCfgAccessCap::new(
            size_of::<VirtioPciCfgAccessCap>() as u8 + PCI_CAP_VNDR_AND_NEXT_SIZE,
            VirtioPciCapType::CfgAccess as u8,
        );
        self.cfg_cap_offset = self.modern_mem_region_map(cfg_cap)?;

        // Make related fields of PCI config writable for VirtioPciCfgAccessCap.
        let write_mask = &mut self.base.config.write_mask[self.cfg_cap_offset..];
        write_mask[offset_of!(VirtioPciCap, bar_id)] = !0;
        le_write_u32(write_mask, offset_of!(VirtioPciCap, offset), !0)?;
        le_write_u32(write_mask, offset_of!(VirtioPciCap, length), !0)?;
        le_write_u32(
            write_mask,
            offset_of!(VirtioPciCfgAccessCap, pci_cfg_data),
            !0,
        )?;

        let nvectors = self.device.lock().unwrap().queue_num() + 1;
    	// 初始化init_msix
        init_msix(
            &mut self.base,
            VIRTIO_PCI_MSIX_BAR_IDX as usize,
            nvectors as u32,
            self.dev_id.clone(),
            None,
            None,
        )?;

        init_intx(
            self.name(),
            &mut self.base.config,
            self.base.parent_bus.clone(),
            self.base.devfn,
        )?;
		// 注册中断回调函数（在complete_one_request中会被调用，判断需不需要notify的时候。
        self.assign_interrupt_cb();

        #[cfg(feature = "virtio_gpu")]
        if device_quirk == Some(VirtioDeviceQuirk::VirtioGpuEnableBar0) {
            init_gpu_bar0(&self.device, &mut self.base.config)?;
        }
		// 初始化virtio-blk
        self.device
            .lock()
            .unwrap()
            .realize()
            .with_context(|| "Failed to realize virtio device")?;

        let name = self.name();
        let devfn = self.base.devfn;
        let dev = Arc::new(Mutex::new(self));
        let mut mem_region_size = ((VIRTIO_PCI_CAP_NOTIFY_OFFSET + VIRTIO_PCI_CAP_NOTIFY_LENGTH)
            as u64)
            .next_power_of_two();
        mem_region_size = max(mem_region_size, MINIMUM_BAR_SIZE_FOR_MMIO as u64);
        let modern_mem_region =
            Region::init_container_region(mem_region_size, "VirtioPciModernMem");
        // 这一步会对各种cap真实所在区域的读写op进行初始化，使得之后如果guest访问这片io区域会trap
    	Self::modern_mem_region_init(dev.clone(), &modern_mem_region)?;

        dev.lock().unwrap().base.config.register_bar(
            VIRTIO_PCI_MEM_BAR_IDX as usize,
            modern_mem_region,
            RegionType::Mem64Bit,
            false,
            mem_region_size,
        )?;

        // Register device to pci bus.
        let pci_bus = dev.lock().unwrap().base.parent_bus.upgrade().unwrap();
        let mut locked_pci_bus = pci_bus.lock().unwrap();
        let pci_device = locked_pci_bus.devices.get(&devfn);
        if pci_device.is_none() {
            locked_pci_bus.devices.insert(devfn, dev.clone());
        } else {
            bail!(
                "Devfn {:?} has been used by {:?}",
                &devfn,
                pci_device.unwrap().lock().unwrap().name()
            );
        }

        MigrationManager::register_transport_instance(VirtioPciState::descriptor(), dev, &name);

        Ok(())
    }
```
## init_msix
```rust
// 解析msix信息，创建msix table和pba
pub fn init_msix(
    pcidev_base: &mut PciDevBase,
    bar_id: usize,
    vector_nr: u32,
    dev_id: Arc<AtomicU16>,
    parent_region: Option<&Region>,
    offset_opt: Option<(u32, u32)>,
) -> Result<()> {
    let config = &mut pcidev_base.config;
    let parent_bus = &pcidev_base.parent_bus;
    if vector_nr == 0 || vector_nr > MSIX_TABLE_SIZE_MAX as u32 + 1 {
        bail!(
            "invalid msix vectors, which should be in [1, {}]",
            MSIX_TABLE_SIZE_MAX + 1
        );
    }

    let msix_cap_offset: usize = config.add_pci_cap(CapId::Msix as u8, MSIX_CAP_SIZE as usize)?;
    let mut offset: usize = msix_cap_offset + MSIX_CAP_CONTROL as usize;
    le_write_u16(&mut config.config, offset, vector_nr as u16 - 1)?;
    le_write_u16(
        &mut config.write_mask,
        offset,
        MSIX_CAP_FUNC_MASK | MSIX_CAP_ENABLE,
    )?;
    offset = msix_cap_offset + MSIX_CAP_TABLE as usize;
    let table_size = vector_nr * MSIX_TABLE_ENTRY_SIZE as u32;
    let pba_size = ((round_up(vector_nr as u64, 64).unwrap() / 64) * 8) as u32;
    let (table_offset, pba_offset) = offset_opt.unwrap_or((0, table_size));
    if ranges_overlap(
        table_offset as usize,
        table_size as usize,
        pba_offset as usize,
        pba_size as usize,
    )
    .unwrap()
    {
        bail!("msix table and pba table overlapped.");
    }
    le_write_u32(&mut config.config, offset, table_offset | bar_id as u32)?;
    offset = msix_cap_offset + MSIX_CAP_PBA as usize;
    le_write_u32(&mut config.config, offset, pba_offset | bar_id as u32)?;

    let msi_irq_manager = if let Some(pci_bus) = parent_bus.upgrade() {
        let locked_pci_bus = pci_bus.lock().unwrap();
        locked_pci_bus.get_msi_irq_manager()
    } else {
        error!("Msi irq controller is none");
        None
    };

    let msix = Arc::new(Mutex::new(Msix::new(
        table_size,
        pba_size,
        msix_cap_offset as u16,
        dev_id.clone(),
        msi_irq_manager,
    )));
    if let Some(region) = parent_region {
        Msix::register_memory_region(
            msix.clone(),
            region,
            dev_id,
            table_offset as u64,
            pba_offset as u64,
        )?;
    } else {
        let mut bar_size = ((table_size + pba_size) as u64).next_power_of_two();
        bar_size = max(bar_size, MINIMUM_BAR_SIZE_FOR_MMIO as u64);
        let region = Region::init_container_region(bar_size, "Msix_region");
        Msix::register_memory_region(
            msix.clone(),
            &region,
            dev_id,
            table_offset as u64,
            pba_offset as u64,
        )?;
        config.register_bar(bar_id, region, RegionType::Mem32Bit, false, bar_size)?;
    }

    config.msix = Some(msix.clone());

    #[cfg(not(test))]
    MigrationManager::register_device_instance(MsixState::descriptor(), msix, &pcidev_base.base.id);

    Ok(())
}

// msix register_memory_region
fn register_memory_region(
        msix: Arc<Mutex<Self>>,
        region: &Region,
        dev_id: Arc<AtomicU16>,
        table_offset: u64,
        pba_offset: u64,
    ) -> Result<()> {
        let locked_msix = msix.lock().unwrap();
        let table_size = locked_msix.table.len() as u64;
        let pba_size = locked_msix.pba.len() as u64;

        let cloned_msix = msix.clone();
        //读table
        let table_read = move |data: &mut [u8], _addr: GuestAddress, offset: u64| -> bool {
            if offset as usize + data.len() > cloned_msix.lock().unwrap().table.len() {
                error!(
                    "It's forbidden to read out of the msix table(size: {}), with offset of {} and size of {}",
                    cloned_msix.lock().unwrap().table.len(),
                    offset,
                    data.len()
                );
                return false;
            }
            let offset = offset as usize;
            data.copy_from_slice(&cloned_msix.lock().unwrap().table[offset..(offset + data.len())]);
            true
        };
        let cloned_msix = msix.clone();
        // 写Table,最后需要更新irq(update_irq_routing)
        let table_write = move |data: &[u8], _addr: GuestAddress, offset: u64| -> bool {
            if offset as usize + data.len() > cloned_msix.lock().unwrap().table.len() {
                error!(
                    "It's forbidden to write out of the msix table(size: {}), with offset of {} and size of {}",
                    cloned_msix.lock().unwrap().table.len(),
                    offset,
                    data.len()
                );
                return false;
            }
            let mut locked_msix = cloned_msix.lock().unwrap();
            // 这里vector是这个msix在msix table中的偏移
            let vector: u16 = offset as u16 / MSIX_TABLE_ENTRY_SIZE;
            let was_masked: bool = locked_msix.is_vector_masked(vector);
            let offset = offset as usize;
            locked_msix.table[offset..(offset + 4)].copy_from_slice(data);

            let is_masked: bool = locked_msix.is_vector_masked(vector);
            if was_masked != is_masked && locked_msix.update_irq_routing(vector, is_masked).is_err()
            {
                return false;
            }

            // Clear the pending vector just when it is pending. Otherwise, it
            // will cause unknown error.
            if was_masked && !is_masked && locked_msix.is_vector_pending(vector) {
                locked_msix.clear_pending_vector(vector);
                locked_msix.notify(vector, dev_id.load(Ordering::Acquire));
            }

            true
        };
        // 注册table pba读写的ops
        let table_region_ops = RegionOps {
            read: Arc::new(table_read),
            write: Arc::new(table_write),
        };
        let table_region = Region::init_io_region(table_size, table_region_ops, "MsixTable");
        region
            .add_subregion(table_region, table_offset)
            .with_context(|| "Failed to register MSI-X table region.")?;

        let cloned_msix = msix.clone();
        let pba_read = move |data: &mut [u8], _addr: GuestAddress, offset: u64| -> bool {
            if offset as usize + data.len() > cloned_msix.lock().unwrap().pba.len() {
                error!(
                    "Fail to read msi pba, illegal data length {}, offset {}",
                    data.len(),
                    offset
                );
                return false;
            }
            let offset = offset as usize;
            data.copy_from_slice(&cloned_msix.lock().unwrap().pba[offset..(offset + data.len())]);
            true
        };
        let pba_write = move |_data: &[u8], _addr: GuestAddress, _offset: u64| -> bool { true };
        let pba_region_ops = RegionOps {
            read: Arc::new(pba_read),
            write: Arc::new(pba_write),
        };
        let pba_region = Region::init_io_region(pba_size, pba_region_ops, "MsixPba");
        region
            .add_subregion(pba_region, pba_offset)
            .with_context(|| "Failed to register MSI-X PBA region.")?;

        Ok(())
    }
	fn update_irq_routing(&mut self, vector: u16, is_masked: bool) -> Result<()> {
        // 当前的entry
        let entry = self.get_message(vector);
/*/// GSI information for routing msix.
struct GsiMsiRoute {
    irq_fd: Arc<EventFd>,
    gsi: i32,
    msg: Message,
}*/
        // 这个vector对应的route，gsi_msi_routes在activate_device中!self.queues_register_irqfd(&call_evts.events)更新，但如果设备need_irqfd为false则不需要，这一步直接返回
        let route = if let Some(route) = self.gsi_msi_routes.get_mut(&vector) {
            route
        } else {
            return Ok(());
        };

        let msix_vector = MsiVector {
            msg_addr_lo: entry.address_lo,
            msg_addr_hi: entry.address_hi,
            msg_data: entry.data,
            masked: false,
            #[cfg(target_arch = "aarch64")]
            dev_id: self.dev_id.load(Ordering::Acquire) as u32,
        };

        let irq_manager = self.msi_irq_manager.as_ref().unwrap();
		// 如果masked就不要注册这个irq
        if is_masked {
            irq_manager.unregister_irqfd(route.irq_fd.clone(), route.gsi as u32)?;
        } else {
            let msg = &route.msg;
            // 如果route信息不同了就更新路由表
            if msg.data != entry.data
                || msg.address_lo != entry.address_lo
                || msg.address_hi != entry.address_hi
            {
                irq_manager.update_route_table(route.gsi as u32, msix_vector)?;
                route.msg = entry;
            }
			// 最终是调用kvm_ioctls::ioctls::vm::VmFd,是kvm的库.在activate_device的时候会更新msix gsi的map,并且向kvm注册irqfd.activate_device在write_common_config被调用,build_common_cfg_ops中初始化设置write_common_config是common region的RegionOps(write和read都需要初始化)
            irq_manager.register_irqfd(route.irq_fd.clone(), route.gsi as u32)?;
        }
        Ok(())
    }
```
## assign_interrupt_cb: 为了注册回调函数
```rust
 fn assign_interrupt_cb(&mut self) {
        let locked_dev = self.device.lock().unwrap();
        let virtio_base = locked_dev.virtio_base();
        let device_status = virtio_base.device_status.clone();
        let interrupt_status = virtio_base.interrupt_status.clone();
        let msix_config = virtio_base.config_vector.clone();
        let config_generation = virtio_base.config_generation.clone();

        let cloned_msix = self.base.config.msix.as_ref().unwrap().clone();
        let cloned_intx = self.base.config.intx.as_ref().unwrap().clone();
        let dev_id = self.dev_id.clone();

        let cb = Arc::new(Box::new(
            move |int_type: &VirtioInterruptType, queue: Option<&Queue>, needs_reset: bool| {
                // vector对应是设config virtio还是vring的interrupt,vring的interrupt中一个queue对应有一个interrupt
                let vector = match int_type {
                    VirtioInterruptType::Config => {
                        if needs_reset {
                            device_status.fetch_or(CONFIG_STATUS_NEEDS_RESET, Ordering::SeqCst);
                        }
                        if device_status.load(Ordering::Acquire) & CONFIG_STATUS_DRIVER_OK == 0 {
                            return Ok(());
                        }

                        // Use (CONFIG | VRING) instead of CONFIG, it can be used to solve the
                        // IO stuck problem by change the device configure.
                        interrupt_status.fetch_or(
                            VIRTIO_MMIO_INT_CONFIG | VIRTIO_MMIO_INT_VRING,
                            Ordering::SeqCst,
                        );
                        config_generation.fetch_add(1, Ordering::SeqCst);
                        // config中断
                        msix_config.load(Ordering::Acquire)
                    }
                    VirtioInterruptType::Vring => {
                        interrupt_status.fetch_or(VIRTIO_MMIO_INT_VRING, Ordering::SeqCst);
                        // queue中断
                        queue.map_or(0, |q| q.vring.get_queue_config().vector)
                    }
                };
/*pub struct QueueConfig {
    /// Guest physical address of the descriptor table.
    pub desc_table: GuestAddress,
    /// Guest physical address of the available ring.
    pub avail_ring: GuestAddress,
    /// Guest physical address of the used ring.
    pub used_ring: GuestAddress,
    /// Host address cache.
    pub addr_cache: VirtioAddrCache,
    /// The maximal size of elements offered by the device.
    pub max_size: u16,
    /// The queue size set by the guest.
    pub size: u16,
    /// Virtual queue ready bit.
    pub ready: bool,
    /// Interrupt vector index of the queue for msix
    pub vector: u16,
    /// The next index which can be popped in the available vring.
    next_avail: Wrapping<u16>,
    /// The next index which can be pushed in the used vring.
    next_used: Wrapping<u16>,
    /// The index of last descriptor used which has triggered interrupt.
    last_signal_used: Wrapping<u16>,
    /// The last_signal_used is valid or not.
    signal_used_valid: bool,
}*/
                let mut locked_msix = cloned_msix.lock().unwrap();
                if locked_msix.enabled {
                    // notify会先判断中断合法不,然后调用self.send_msix(vector, dev_id);最终会调用kvm的signal_msi：self.vm_fd.signal_msi(kvm_msi);
                    locked_msix.notify(vector, dev_id.load(Ordering::Acquire));
                } else {
                    cloned_intx.lock().unwrap().notify(1);
                }

                Ok(())
            },
        ) as VirtioInterrupt);

        self.interrupt_cb = Some(cb);
    }

	pub fn send_msix(&self, vector: u16, dev_id: u16) {
        // 通过vector在msix table中获取信息(address和data)
        let msg = self.get_message(vector);

        if is_test_enabled() {
            let data = msg.data;
            let mut addr: u64 = msg.address_hi as u64;
            addr = (addr << 32) + msg.address_lo as u64;
            add_msix_msg(addr, data);
            return;
        }

        let msix_vector = MsiVector {
            msg_addr_lo: msg.address_lo,
            msg_addr_hi: msg.address_hi,
            msg_data: msg.data,
            masked: false,
            #[cfg(target_arch = "aarch64")]
            dev_id: dev_id as u32,
        };

        let irq_manager = self.msi_irq_manager.as_ref().unwrap();
        // trigger在kvm里实现
        if let Err(e) = irq_manager.trigger(None, msix_vector, dev_id as u32) {
            error!("Send msix error: {:?}", e);
        };
    }

	fn trigger(&self, irq_fd: Option<Arc<EventFd>>, vector: MsiVector, dev_id: u32) -> Result<()> {
        if irq_fd.is_some() {	
            irq_fd.unwrap().write(1)?;
        } else {	//virtio-blk-pci没设置，这个分支
            #[cfg(target_arch = "aarch64")]
            let flags: u32 = kvm_bindings::KVM_MSI_VALID_DEVID;
            #[cfg(target_arch = "x86_64")]
            let flags: u32 = 0;

            let kvm_msi = kvm_bindings::kvm_msi {
                address_lo: vector.msg_addr_lo,
                address_hi: vector.msg_addr_hi,
                data: vector.msg_data,
                flags,
                devid: dev_id,
                pad: [0; 12],
            };
			// 是kvm_ioctl的函数
            self.vm_fd.signal_msi(kvm_msi)?;
        }

        Ok(())
    }
```
## modern_mem_region_init
```rust
	//此处传入的virtio_pci就是self对应的设备
	fn modern_mem_region_init(
        virtio_pci: Arc<Mutex<VirtioPciDevice>>,
        modern_mem_region: &Region,
    ) -> Result<()> {
        // 1. PCI common cap sub-region.
        // common cfg ops这里会初始化common region的读写指令,read和write最终会调用read_common_config和write_common_config,其中write_common_config会涉及到初始化virtio的config,从而初始化msix对应的irqfd
        let common_region_ops = Self::build_common_cfg_ops(virtio_pci.clone());
        let common_region = Region::init_io_region(
            u64::from(VIRTIO_PCI_CAP_COMMON_LENGTH),
            common_region_ops,
            "VirtioPciCommon",
        );
        modern_mem_region
            .add_subregion(common_region, u64::from(VIRTIO_PCI_CAP_COMMON_OFFSET))
            .with_context(|| "Failed to register pci-common-cap region.")?;

        // 2. PCI ISR cap sub-region.
        let cloned_device = virtio_pci.lock().unwrap().device.clone();
        let cloned_intx = virtio_pci.lock().unwrap().base.config.intx.clone().unwrap();
        let isr_read = move |data: &mut [u8], _: GuestAddress, _: u64| -> bool {
            if let Some(val) = data.get_mut(0) {
                let device_lock = cloned_device.lock().unwrap();
                *val = device_lock
                    .virtio_base()
                    .interrupt_status
                    .swap(0, Ordering::SeqCst) as u8;
                cloned_intx.lock().unwrap().notify(0);
            }
            true
        };
        let isr_write = move |_: &[u8], _: GuestAddress, _: u64| -> bool { true };
        let isr_region_ops = RegionOps {
            read: Arc::new(isr_read),
            write: Arc::new(isr_write),
        };
        let isr_region = Region::init_io_region(
            u64::from(VIRTIO_PCI_CAP_ISR_LENGTH),
            isr_region_ops,
            "VirtioIsr",
        );
        modern_mem_region
            .add_subregion(isr_region, u64::from(VIRTIO_PCI_CAP_ISR_OFFSET))
            .with_context(|| "Failed to register pci-isr-cap region.")?;

        // 3. PCI dev cap sub-region.
        let cloned_virtio_dev = virtio_pci.lock().unwrap().device.clone();
        let device_read = move |data: &mut [u8], _addr: GuestAddress, offset: u64| -> bool {
            if let Err(e) = cloned_virtio_dev.lock().unwrap().read_config(offset, data) {
                error!("Failed to read virtio-dev config space, error is {:?}", e);
                return false;
            }
            true
        };

        let cloned_virtio_dev = virtio_pci.lock().unwrap().device.clone();
        let device_write = move |data: &[u8], _addr: GuestAddress, offset: u64| -> bool {
            if let Err(e) = cloned_virtio_dev.lock().unwrap().write_config(offset, data) {
                error!("Failed to write virtio-dev config space, error is {:?}", e);
                return false;
            }
            true
        };
        let device_region_ops = RegionOps {
            read: Arc::new(device_read),
            write: Arc::new(device_write),
        };
        let device_region = Region::init_io_region(
            u64::from(VIRTIO_PCI_CAP_DEVICE_LENGTH),
            device_region_ops,
            "VirtioDevice",
        );
        modern_mem_region
            .add_subregion(device_region, u64::from(VIRTIO_PCI_CAP_DEVICE_OFFSET))
            .with_context(|| "Failed to register pci-dev-cap region.")?;

        // 4. PCI notify cap sub-region.
        let notify_read = move |_: &mut [u8], _: GuestAddress, _: u64| -> bool { true };
        let notify_write = move |_: &[u8], _: GuestAddress, _: u64| -> bool { true };
        let notify_region_ops = RegionOps {
            read: Arc::new(notify_read),
            write: Arc::new(notify_write),
        };
        let notify_region = Region::init_io_region(
            u64::from(VIRTIO_PCI_CAP_NOTIFY_LENGTH),
            notify_region_ops,
            "VirtioNotify",
        );
        // ioeventfds()类型是RegionIoEventFd，由notify_eventfds得到的。guest向特定的内存地址写入数据时，可以触发一个事件，具体事件应该在block.rs中定义，相当于得用另一个io thread等待这个fd被写？？
    /*pub struct RegionIoEventFd {
    	/// EventFd to be triggered when guest writes to the address.
	    pub fd: Arc<vmm_sys_util::eventfd::EventFd>,
    	/// Addr_range contains two params as follows:
	    /// base: in addr_range is the address of EventFd.
    	/// size: can be 2, 4, 8 bytes.
	    pub addr_range: AddressRange,
    	/// If data_match is enabled.
	    pub data_match: bool,
    	/// The specified value to trigger events.
	    pub data: u64,
   	}
    fn ioeventfds(&self) -> Vec<RegionIoEventFd> {
        let mut ret = Vec::new();
        let eventfds = (*self.notify_eventfds).clone();
        for (index, eventfd) in eventfds.events.into_iter().enumerate() {
            let addr = index as u64 * u64::from(VIRTIO_PCI_CAP_NOTIFY_OFF_MULTIPLIER);
            ret.push(RegionIoEventFd {
                fd: eventfd.clone(),
                addr_range: AddressRange::from((addr, 2u64)),
                data_match: false,
                data: index as u64,
            })
        }

        ret
    }*/
        notify_region.set_ioeventfds(&virtio_pci.lock().unwrap().ioeventfds());

        modern_mem_region
            .add_subregion(notify_region, u64::from(VIRTIO_PCI_CAP_NOTIFY_OFFSET))
            .with_context(|| "Failed to register pci-notify-cap region.")?;

        Ok(())
    }
```
### write_common_config
```rust
fn write_common_config(&mut self, offset: u64, value: u32) -> Result<()> {
        trace::virtio_tpt_write_common_config(&self.base.base.id, offset, value);
        let mut locked_device = self.device.lock().unwrap();
        match offset {
            COMMON_DFSELECT_REG => {
                locked_device.set_hfeatures_sel(value);
            }
            COMMON_GFSELECT_REG => {
                locked_device.set_gfeatures_sel(value);
            }
            COMMON_GF_REG => {
                if locked_device.device_status() & CONFIG_STATUS_FEATURES_OK != 0 {
                    error!("it's not allowed to set features after having been negoiated");
                    return Ok(());
                }
                let gfeatures_sel = locked_device.gfeatures_sel();
                if gfeatures_sel >= MAX_FEATURES_SELECT_NUM {
                    return Err(anyhow!(PciError::FeaturesSelect(gfeatures_sel)));
                }
                locked_device.set_driver_features(gfeatures_sel, value);

                if gfeatures_sel == 1 {
                    let features = (locked_device.driver_features(1) as u64) << 32;
                    if virtio_has_feature(features, VIRTIO_F_RING_PACKED) {
                        locked_device.set_queue_type(QUEUE_TYPE_PACKED_VRING);
                    } else {
                        locked_device.set_queue_type(QUEUE_TYPE_SPLIT_VRING);
                    }
                }
            }
            COMMON_MSIX_REG => {
                if self.base.config.revise_msix_vector(value) {
                    locked_device.set_config_vector(value as u16);
                } else {
                    locked_device.set_config_vector(INVALID_VECTOR_NUM);
                }
                locked_device.set_interrupt_status(0);
            }
            // 对应当前device的status
            COMMON_STATUS_REG => {
                // 先对当前的status进行写
                if value & CONFIG_STATUS_FEATURES_OK != 0 && value & CONFIG_STATUS_DRIVER_OK == 0 {
                    let features = (locked_device.driver_features(1) as u64) << 32;
                    if !virtio_has_feature(features, VIRTIO_F_VERSION_1) {
                        error!(
                            "Device is modern only, but the driver not support VIRTIO_F_VERSION_1"
                        );
                        return Ok(());
                    }
                }
                if value != 0 && (locked_device.device_status() & !value) != 0 {
                    error!("Driver must not clear a device status bit");
                    return Ok(());
                }

                let old_status = locked_device.device_status();
                locked_device.set_device_status(value);
                // 如果是driver ok 了就可以激活设备，说明驱动可以和这个设备交互了
                if locked_device.check_device_status(
                    CONFIG_STATUS_ACKNOWLEDGE
                        | CONFIG_STATUS_DRIVER
                        | CONFIG_STATUS_DRIVER_OK
                        | CONFIG_STATUS_FEATURES_OK,
                    CONFIG_STATUS_FAILED,
                ) {
                    drop(locked_device);
                    self.activate_device();
                } else if old_status != 0 && locked_device.device_status() == 0 {
                    drop(locked_device);
                    self.deactivate_device();
                }
            }
            COMMON_Q_SELECT_REG => {
                if value < VIRTIO_QUEUE_MAX {
                    locked_device.set_queue_select(value as u16);
                }
            }
            COMMON_Q_SIZE_REG => locked_device
                .queue_config_mut(true)
                .map(|config| config.size = value as u16)?,
            COMMON_Q_ENABLE_REG => {
                if value != 1 {
                    error!("Driver set illegal value for queue_enable {}", value);
                    return Err(anyhow!(PciError::QueueEnable(value)));
                }
                locked_device
                    .queue_config_mut(true)
                    .map(|config| config.ready = true)?;
            }
            COMMON_Q_MSIX_REG => {
                let val = if self.base.config.revise_msix_vector(value) {
                    value as u16
                } else {
                    INVALID_VECTOR_NUM
                };
                // It should not check device status when detaching device which
                // will set vector to INVALID_VECTOR_NUM.
                let need_check = locked_device.device_status() != 0;
                locked_device
                    .queue_config_mut(need_check)
                    .map(|config| config.vector = val)?;
            }
            COMMON_Q_DESCLO_REG => locked_device.queue_config_mut(true).map(|config| {
                config.desc_table = GuestAddress(config.desc_table.0 | u64::from(value));
            })?,
            COMMON_Q_DESCHI_REG => locked_device.queue_config_mut(true).map(|config| {
                config.desc_table = GuestAddress(config.desc_table.0 | (u64::from(value) << 32));
            })?,
            COMMON_Q_AVAILLO_REG => locked_device.queue_config_mut(true).map(|config| {
                config.avail_ring = GuestAddress(config.avail_ring.0 | u64::from(value));
            })?,
            COMMON_Q_AVAILHI_REG => locked_device.queue_config_mut(true).map(|config| {
                config.avail_ring = GuestAddress(config.avail_ring.0 | (u64::from(value) << 32));
            })?,
            COMMON_Q_USEDLO_REG => locked_device.queue_config_mut(true).map(|config| {
                config.used_ring = GuestAddress(config.used_ring.0 | u64::from(value));
            })?,
            COMMON_Q_USEDHI_REG => locked_device.queue_config_mut(true).map(|config| {
                config.used_ring = GuestAddress(config.used_ring.0 | (u64::from(value) << 32));
            })?,
            _ => {
                return Err(anyhow!(PciError::PciRegister(offset)));
            }
        };

        Ok(())
    }
```
#### activate_device
```rust
	fn activate_device(&self) -> bool {
        trace::virtio_tpt_common("activate_device", &self.base.base.id);
        let mut locked_dev = self.device.lock().unwrap();
        if locked_dev.device_activated() {
            return true;
        }

        let queue_type = locked_dev.queue_type();
        let features = locked_dev.virtio_base().driver_features;
        let broken = locked_dev.virtio_base().broken.clone();

        let mut queues = Vec::new();
        let queues_config = &mut locked_dev.virtio_base_mut().queues_config;
        for q_config in queues_config.iter_mut() {
            if !q_config.ready {
                debug!("queue is not ready, please check your init process");
            } else {
                q_config.set_addr_cache(
                    self.sys_mem.clone(),
                    self.interrupt_cb.clone().unwrap(),
                    features,
                    &broken,
                );
            }
            let queue = Queue::new(*q_config, queue_type).unwrap();
            if q_config.ready && !queue.is_valid(&self.sys_mem) {
                error!("Failed to activate device: Invalid queue");
                return false;
            }
            let arc_queue = Arc::new(Mutex::new(queue));
            queues.push(arc_queue.clone());
        }
        locked_dev.virtio_base_mut().queues = queues;

        let parent = self.base.parent_bus.upgrade().unwrap();
        parent
            .lock()
            .unwrap()
            .update_dev_id(self.base.devfn, &self.dev_id);
        // 如果需要irqfd在这里分配
        if self.need_irqfd {	
            let mut queue_num = locked_dev.queue_num();
            // No need to create call event for control queue.
            // It will be polled in StratoVirt when activating the device.
            if locked_dev.has_control_queue() && queue_num % 2 != 0 {
                queue_num -= 1;
            }
            let call_evts = NotifyEventFds::new(queue_num);
            if let Err(e) = locked_dev.set_guest_notifiers(&call_evts.events) {
                error!("Failed to set guest notifiers, error is {:?}", e);
                return false;
            }
            drop(locked_dev);
            if !self.queues_register_irqfd(&call_evts.events) {
                error!("Failed to register queues irqfd.");
                return false;
            }
            locked_dev = self.device.lock().unwrap();
        }

        // vmm仓库里的eventfd,每个queue对应一个，共有queue_num个。对应的activate是virtio_blk的
        /*pub struct NotifyEventFds {
    		pub events: Vec<Arc<EventFd>>,
		}*/
        let queue_evts = (*self.notify_eventfds).clone().events;	
        if let Err(e) = 	locked_dev.activate(
            self.sys_mem.clone(),
            self.interrupt_cb.clone().unwrap(),
            queue_evts,
        ) {
            error!("Failed to activate device, error is {:?}", e);
            return false;
        }

        locked_dev.set_device_activated(true);
        true
    }

// virtio/src/device/block.rs

    fn activate(
        &mut self,
        mem_space: Arc<AddressSpace>,
        interrupt_cb: Arc<VirtioInterrupt>,
        queue_evts: Vec<Arc<EventFd>>,
    ) -> Result<()> {
        self.interrupt_cb = Some(interrupt_cb.clone());
        let queues = self.base.queues.clone();
        for (index, queue) in queues.iter().enumerate() {
            if !queue.lock().unwrap().is_enabled() {
                continue;
            }
            let (sender, receiver) = channel();
            let update_evt = Arc::new(EventFd::new(libc::EFD_NONBLOCK)?);
            let driver_features = self.base.driver_features;
            let handler = BlockIoHandler {
                queue: queue.clone(),
                queue_evt: queue_evts[index].clone(),
                mem_space: mem_space.clone(),
                block_backend: self.block_backend.clone(),
                req_align: self.req_align,
                buf_align: self.buf_align,
                disk_sectors: self.disk_sectors,
                direct: self.blk_cfg.direct,
                serial_num: self.blk_cfg.serial_num.clone(),
                driver_features,
                receiver,
                update_evt: update_evt.clone(),
                device_broken: self.base.broken.clone(),
                interrupt_cb: interrupt_cb.clone(),	// 后续应该和队列处理有关系
                iothread: self.blk_cfg.iothread.clone(),
                leak_bucket: match self.blk_cfg.iops {
                    Some(iops) => Some(LeakBucket::new(iops)?),
                    None => None,
                },
                discard: self.blk_cfg.discard,
                write_zeroes: self.blk_cfg.write_zeroes,
            };

            let notifiers = EventNotifierHelper::internal_notifiers(Arc::new(Mutex::new(handler)));
            register_event_helper(
                notifiers,
                self.blk_cfg.iothread.as_ref(),
                &mut self.base.deactivate_evts,
            )?;
            self.update_evts.push(update_evt);
            self.senders.push(sender);
        }
```



virtio pci capability 种类

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191222174524659.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1YW5nOTg3MjQ2NTEw,size_16,color_FFFFFF,t_70)

前端通知后端：写notify cfg吧

后端通知前端：利用kvm库发送msix