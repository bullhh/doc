Provided by @[TSM](https://github.com/915604903T) 

# device model
![../../_images/virtio-hld-image3.png](https://projectacrn.github.io/latest/_images/virtio-hld-image3.png)
主要关注device model部分，但是device model是user space，service vm相当于是一个异步处理vmexit的线程？？处理完了之后再通过hypercall陷入到hypervisor，让hypervisor告诉user vm。所以dm 里面的vm相当于只是一个shadow vm？？主要关注怎么模拟virtio-pci的交互，还有dm->hypervisor->user vm的msix是怎么注入的吧。
应该是dm作为守护虚拟机运行在用户态，和hypervisor用ioctl？？？

在acrn vmexit.c中定义了dispatch_table，在vmcall.c中定义了hc_dispatch_table，前者是vmexit_reason，后者是根据不同hypercall的参数，hc_dispatch_table在dispatch_hypercall->vmcall_vmexit_handler（为dispatch_table中定义的一项）中被调用。

相当于hsm是连接hypervisor和dm的工具。每个队列有一个ioeventfd，应该会被kernel kick有io到来。ept_violation_vmexit_handler/pio_instr_vmexit_handler中都会调用emulate_io，对于可能没有在hv中注册的mmio或者pio的地址，就会调用`int32_t acrn_insert_request(struct acrn_vcpu *vcpu, const struct io_request *io_req)`其中io_request最早是从vcpu来的。
- acrn_insert_request：
  1. 检查 vcpu 的虚拟机是否有共享的 I/O 页面，并且当前 vcpu 的 I/O 请求状态是否为空闲（ACRN_IOREQ_STATE_FREE）。如果不满足这些条件，函数将返回 -EINVAL。
  2. 如果满足上述条件，函数将获取指向 I/O 请求缓冲区的指针，并获取当前 vcpu 的 ID
  3. 然后，函数将复制 io_req 的内容到 I/O 请求缓冲区的相应位置。如果虚拟机处于 I/O 请求轮询模式，函数将设置 acrn_io_req 的 completion_polling 字段，并将 is_polling 设置为 true。
  4. 函数将清除 I/O 请求的信号，并将其状态设置为挂起（ACRN_IOREQ_STATE_PENDING）。然后，函数将触发一个中断，通知 hsm，这个中断应该是一开始在hsm中注册的。
  5. 如果虚拟机不处于 I/O 请求轮询模式，函数将等待 I/O 请求事件的完成。
  其中共享页面是通过hypercall（hcall_set_ioreq_buffer）设置的，这个handler是在hsm初始化ioreq的时候（acrn_ioreq_init）调用hcall_set_ioreq_buffer发送id是HC_SET_IOREQ_BUFFER的hypercall时被调用（`return acrn_hypercall2(HC_SET_IOREQ_BUFFER, vmid, buffer);`）这个函数的buffer是在acrn_ioreq_init中分配的内核内存，acrn_ioreq_init在acrn_vm_creation被调用，acrn_vm_creation是在dm发送ACRN_IOCTL_CREATE_VM ioctl搞出来的，这个ioctl在vm_create中被调用
```rust
// buf_vma是由acrn_vm_create函数中struct acrn_vm_creation *vm_param参数中的ioreq_buf传来的。
int acrn_ioreq_init(struct acrn_vm *vm, u64 buf_vma)
{
	struct acrn_ioreq_buffer *set_buffer;
	struct page *page;
	int ret;

	if (vm->ioreq_buf)
		return -EEXIST;

	set_buffer = kzalloc(sizeof(*set_buffer), GFP_KERNEL);
	if (!set_buffer)
		return -ENOMEM;
	// pin_user_pages_fast函数被用来获取用户空间地址buf_vma对应的页框，并将其锁定在内存中，防止其被换出。这个函数返回锁定的页框数量，并将页框的指针存储在page变量中
	ret = pin_user_pages_fast(buf_vma, 1,
				  FOLL_WRITE | FOLL_LONGTERM, &page);
	if (unlikely(ret != 1) || !page) {
		dev_err(acrn_dev.this_device, "Failed to pin ioreq page!\n");
		ret = -EFAULT;
		goto free_buf;
	}

	vm->ioreq_buf = page_address(page);
	vm->ioreq_page = page;
	set_buffer->ioreq_buf = page_to_phys(page);
	ret = hcall_set_ioreq_buffer(vm->vmid, virt_to_phys(set_buffer));
	if (ret < 0) {
		dev_err(acrn_dev.this_device, "Failed to init ioreq buffer!\n");
		unpin_user_page(page);
		vm->ioreq_buf = NULL;
		goto free_buf;
	}

	dev_dbg(acrn_dev.this_device,
		"Init ioreq buffer %pK!\n", vm->ioreq_buf);
	ret = 0;
free_buf:
	kfree(set_buffer);
	return ret;
}
```

![../../_images/virtio-hld-image4.png](https://projectacrn.github.io/latest/_images/virtio-hld-image4.png)

## pci emulation

![../../_images/dm-image83.png](https://projectacrn.github.io/latest/_images/dm-image83.png)
PCI Init: PCI initialization scans the PCI bus/slot/function to identify each configured PCI device on the acrn-dm command line and **initializes their configuration space by calling their dedicated vdev_init() function**. For more details on the DM PCI emulation, refer to PCI Emulation.

pci_generate_msix会产生一个msix，最终通过ioctl注入到guest（`error = ioctl(ctx->fd, ACRN_IOCTL_INJECT_MSI, &msi);`具体ioctl过程呢？？），这个函数在vq_interrupt(队列需要发中断)或者virtio_config_changed（virto config改变）的时候被调用。其中config change在rescan被调用，vq_interrupt在vq_endchains被调用（在处理完虚拟队列（vq）中的"available"链表后被调用），vq_endchains在virtio_blk_abort/virtio_blk_done被调用。
**调用流程：virtio_blk_done(处理virtio时被调用) -> vq_endchains -> vq_interrupt -> pci_generate_msix** 经过hsm 最后调用**hcall_inject_msi**，用apic注入

## 执行流

- int **main**(int argc, char *argv[])
  - 在main函数中会调用add_cpu，其中会创建start_thread线程，start_thread中会运行`vm_loop(mtp->mt_ctx);`，
```rust
static void
vm_loop(struct vmctx *ctx)
{
	int error;

	ctx->ioreq_client = vm_create_ioreq_client(ctx);
	if (ctx->ioreq_client < 0) {
		pr_err("%s, failed to create IOREQ.\n", __func__);
		return;
	}

	if (vm_run(ctx) != 0) {
		pr_err("%s, failed to run VM.\n", __func__);
		return;
	}

	while (1) {
		int vcpu_id;
		struct acrn_io_request *io_req;

		error = vm_attach_ioreq_client(ctx);
		if (error)
			break;

		for (vcpu_id = 0; vcpu_id < guest_ncpus; vcpu_id++) {
			io_req = &ioreq_buf[vcpu_id];
			if ((atomic_load(&io_req->processed) == ACRN_IOREQ_STATE_PROCESSING)
				&& !io_req->kernel_handled)
				handle_vmexit(ctx, io_req, vcpu_id);
		}

		if (VM_SUSPEND_FULL_RESET == vm_get_suspend_mode() ||
		    VM_SUSPEND_POWEROFF == vm_get_suspend_mode()) {
			break;
		}

		/* RTVM can't be reset */
		if ((VM_SUSPEND_SYSTEM_RESET == vm_get_suspend_mode()) && (!is_rtvm)) {
			vm_system_reset(ctx);
		}

		if (VM_SUSPEND_SUSPEND == vm_get_suspend_mode()) {
			vm_suspend_resume(ctx);
		}
	}
	pr_err("VM loop exit\n");
}
```
  - vm_loop在start虚拟机后的循环中会调用vm_attach_ioreq_client发送ioctl ACRN_IOCTL_ATTACH_IOREQ_CLIENT对应linux acrn_ioreq_client_wait，等待是否有ioreqs_map pending req，有的话就会返回了。返回就知道ioreq_buf有东西了，可以进行处理这边是vq注册了一个ioeventfd，被kick的时候知道有io请求到来
```rust
/**
 * struct acrn_ioreq_client - Structure of I/O client.
 * @name:	Client name
 * @vm:		The VM that the client belongs to
 * @list:	List node for this acrn_ioreq_client
 * @is_default:	If this client is the default one
 * @flags:	Flags (ACRN_IOREQ_CLIENT_*)
 * @range_list:	I/O ranges
 * @range_lock:	Lock to protect range_list
 * @ioreqs_map:	The pending I/O requests bitmap.
 * @handler:	I/O requests handler of this client
 * @thread:	The thread which executes the handler
 * @wq:		The wait queue for the handler thread parking
 * @priv:	Data for the thread
 */
struct acrn_ioreq_client {
	char			name[ACRN_NAME_LEN];
	struct acrn_vm		*vm;
	struct list_head	list;
	bool			is_default;
	unsigned long		flags;
	struct list_head	range_list;
	rwlock_t		range_lock;
	DECLARE_BITMAP(ioreqs_map, ACRN_IO_REQUEST_MAX);
	ioreq_handler_t		handler;
	struct task_struct	*thread;
	wait_queue_head_t	wq;
	void			*priv;
};
```
  - pci_parse_slot(optarg) != 0	函数定义在devicemodel/hw/pci/core.c中
```rust
    int
    pci_parse_slot(char *opt)
    {
	  struct businfo *bi;
	  struct slotinfo *si;
	  char *emul, *config, *str, *cp, *b = NULL;
	  int error, bnum, snum, fnum;

	error = -1;
	str = strdup(opt);
	if (!str) {
		pr_err("%s: strdup returns NULL\n", __func__);
		return -1;
	}
	// emul: 设备名称，config：配置，str: bdf
	emul = config = NULL;
	cp = str;
	str = strsep(&cp, ",");
	if (cp) {
		emul = strsep(&cp, ",");
		/* for boot device */
		if (cp && *cp == 'b' && *(cp+1) == ',')
			b = strsep(&cp, ",");
		config = cp;
	} else {
		pci_parse_slot_usage(opt);
		goto done;
	}

	if ((strcmp("pci-gvt", emul) == 0) || (strcmp("virtio-hdcp", emul) == 0)
			|| (strcmp("npk", emul) == 0) || (strcmp("virtio-coreu", emul) == 0)) {
		pr_warn("The \"%s\" parameter is obsolete and ignored\n", emul);
		goto done;
	}

	/* <bus>:<slot>:<func> */
	if (parse_bdf(str, &bnum, &snum, &fnum, 10) != 0)
		snum = -1;

	if (bnum < 0 || bnum >= MAXBUSES || snum < 0 || snum >= MAXSLOTS ||
	    fnum < 0 || fnum >= MAXFUNCS) {
		pci_parse_slot_usage(opt);
		goto done;
	}
	// 如果没有分配对应bnum信息的内存，则分配
	if (pci_businfo[bnum] == NULL)
		pci_businfo[bnum] = calloc(1, sizeof(struct businfo));
  /*struct businfo {
	uint16_t iobase, iolimit;		// I/O window 
	uint32_t membase32, memlimit32;		// mmio window below 4GB 
	uint64_t membase64, memlimit64;		// mmio window above 4GB 
	struct slotinfo slotinfo[MAXSLOTS];
  };
  struct slotinfo {
	struct intxinfo si_intpins[4];
	struct funcinfo si_funcs[MAXFUNCS];
  };
  // 中断信息？？
  struct intxinfo {
	int	ii_count;
	int	ii_pirq_pin;
	int	ii_ioapic_irq;
  };
  struct funcinfo {
	char	*fi_name;
	char	*fi_param;
	char	*fi_param_saved; // save for reboot 
	struct pci_vdev *fi_devi;
  };*/
	bi = pci_businfo[bnum];
	si = &bi->slotinfo[snum];

	if (si->si_funcs[fnum].fi_name != NULL) {
		pr_err("pci slot %d:%d already occupied!\n",
			snum, fnum);
		goto done;
	}
  // 会找出对应的pci设备，具体设备定义在devicemodel中，比如virtio_blk:
  /*struct pci_vdev_ops pci_ops_virtio_blk = {
	.class_name	= "virtio-blk",
	.vdev_init	= virtio_blk_init,
	.vdev_deinit	= virtio_blk_deinit,
	.vdev_barwrite	= virtio_pci_write,
	.vdev_barread	= virtio_pci_read
  };
  DEFINE_PCI_DEVTYPE(pci_ops_virtio_blk);	//会把这个设备加入到pci_vdev_ops_set中，pci_emul_finddev就是在遍历pci_vdev_ops_set
  */
	if (pci_emul_finddev(emul) == NULL) {
		pr_err("pci slot %d:%d: unknown device \"%s\"\n",
			snum, fnum, emul);
		goto done;
	}

	error = 0;
	si->si_funcs[fnum].fi_name = emul;
	/* saved fi param in case reboot */
	si->si_funcs[fnum].fi_param_saved = config;

	if (strcmp("virtio-net", emul) == 0) {
		si->si_funcs[fnum].fi_param_saved = cp;
	}
	// 如果是blk是boot blk的处理，vsbl_set_bdf用于设置boot_blk_bdf
	if (b != NULL) {
		if ((strcmp("virtio-blk", emul) == 0) &&  (b != NULL) &&
			(strchr(b, 'b') != NULL)) {
			vsbl_set_bdf(bnum, snum, fnum);
		}
	}

	if ((strcmp("virtio-gpu", emul) == 0)) {
		pr_info("%s: virtio-gpu device found, activating virtual display.\n",
				__func__);
		gfx_ui = true;
		vdpy_parse_cmd_option(config);
	}
  done:
	if (error)
		free(str);

	return error;
  }
```
  - static int vm_init_vdevs(struct vmctx *ctx)
    - int init_pci(struct vmctx *ctx) ：会获取pci的mem区域，从每个bus遍历每个bdf，其中会调用pci_emul_init进行具体设备的初始化
     - static int pci_emul_init(struct vmctx *ctx, struct pci_vdev_ops *ops, int bus, int slot, int func, struct funcinfo *fi)：在这个语句会调用定义的每个设备定义的init函数 err = (*ops->vdev_init)(ctx, pdi, fi->fi_param);以blk为例是virtio_blk_init

## 函数详解
### virtio_blk_init
```rust
static int
virtio_blk_init(struct vmctx *ctx, struct pci_vdev *dev, char *opts)
{
	bool dummy_bctxt;
	char bident[16];
	struct blockif_ctxt *bctxt;
	char *opts_tmp = NULL;
	char *opts_start = NULL;
	char *opt = NULL;
	u_char digest[16];
	struct virtio_blk *blk;
	bool use_iothread;
	int i;
	pthread_mutexattr_t attr;
	int rc;

	bctxt = NULL;
	/* Assume the bctxt is valid, until identified otherwise */
	dummy_bctxt = false;
	use_iothread = false;

	if (opts == NULL) {
		pr_err("virtio_blk: backing device required\n");
		return -1;
	}

	/*
	 * The supplied backing file has to exist
	 */
	if (snprintf(bident, sizeof(bident), "%d:%d",
				dev->slot, dev->func) >= sizeof(bident)) {
		WPRINTF(("bident error, please check slot and func\n"));
	}

	/*
	 * If "nodisk" keyword is found in opts, this is not a valid backend
	 * file. Skip blockif_open and set dummy bctxt in virtio_blk struct
	 */
	
	opts_start = opts_tmp = strdup(opts);
	if (!opts_start) {
		WPRINTF(("%s: strdup failed\n", __func__));
		return -1;
	}
    // 如果没有找到"nodisk"，则继续处理后端文件选项
	if (strstr(opts, "nodisk") == NULL) {
		opt = strsep(&opts_tmp, ",");
		if (strcmp("iothread", opt) == 0) {
			use_iothread = true;
		} else {
			/* The opts_start is truncated by strsep, opts_tmp is also
			 * changed by strsetp, so use opts which points to the
			 * original parameter string
			 */
			opts_tmp = opts;
		}
		bctxt = blockif_open(opts_tmp, bident);
		if (bctxt == NULL) {
			pr_err("Could not open backing file");
			free(opts_start);
			return -1;
		}
	} else {
		dummy_bctxt = true;
	}

	free(opts_start);

	// 分配virtio的blk
	blk = calloc(1, sizeof(struct virtio_blk));
	if (!blk) {
		WPRINTF(("virtio_blk: calloc returns NULL\n"));
		return -1;
	}
	
    // 设置后端文件
	blk->bc = bctxt;
	/* Update virtio-blk device struct of dummy ctxt*/
	blk->dummy_bctxt = dummy_bctxt;
	// 对于每一个virtio vring（包含desc table, used, available??），
	for (i = 0; i < VIRTIO_BLK_RINGSZ; i++) {
		struct virtio_blk_ioreq *io = &blk->ios[i];
        // 设置回调函数,处理了io之后用
		io->req.callback = virtio_blk_done;
		io->req.param = io;
		io->blk = blk;
		io->idx = i;
	}

	/* init mutex attribute properly to avoid deadlock */
	rc = pthread_mutexattr_init(&attr);
	if (rc)
		DPRINTF(("mutexattr init failed with erro %d!\n", rc));
	rc = pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);
	if (rc)
		DPRINTF(("virtio_blk: mutexattr_settype failed with "
					"error %d!\n", rc));

	rc = pthread_mutex_init(&blk->mtx, &attr);
	if (rc)
		DPRINTF(("virtio_blk: pthread_mutex_init failed with "
					"error %d!\n", rc));

	/* init virtio struct and virtqueues */
	virtio_linkup(&blk->base, &virtio_blk_ops, blk, dev, &blk->vq, BACKEND_VBSU);
	blk->base.iothread = use_iothread;
	blk->base.mtx = &blk->mtx;

	blk->vq.qsize = VIRTIO_BLK_RINGSZ;
	/* blk->vq.vq_notify = we have no per-queue notify */
	// 省略了给后端文件建立backup的md5过程
	
	rc = snprintf(blk->ident, sizeof(blk->ident),
		"ACRN--%02X%02X-%02X%02X-%02X%02X", digest[0],
		digest[1], digest[2], digest[3], digest[4], digest[5]);
	if (rc >= sizeof(blk->ident) || rc < 0)
		WPRINTF(("virtio_blk: device name is invalid!\n"));

	/* Setup virtio block config space only for valid backend file*/
	// 更新文件的一些信息？主要是和块设备相关的参数，如大小、扇区大小等信息。
	if (!blk->dummy_bctxt)
		virtio_blk_update_config_space(blk);

	/*
	 * Should we move some of this into virtio.c?  Could
	 * have the device, class, and subdev_0 as fields in
	 * the virtio constants structure.
	 */
	 // 写pci config space，作为以后可以模拟读取的部分？
	pci_set_cfgdata16(dev, PCIR_DEVICE, VIRTIO_DEV_BLOCK);
	pci_set_cfgdata16(dev, PCIR_VENDOR, VIRTIO_VENDOR);
	pci_set_cfgdata8(dev, PCIR_CLASS, PCIC_STORAGE);
	pci_set_cfgdata16(dev, PCIR_SUBDEV_0, VIRTIO_TYPE_BLOCK);
	if (is_winvm == true)
		pci_set_cfgdata16(dev, PCIR_SUBVEND_0, ORACLE_VENDOR_ID);
	else
		pci_set_cfgdata16(dev, PCIR_SUBVEND_0, VIRTIO_VENDOR);
	// 初始化msxi，默认用msix，除非一开始设置说用msi，这个函数最终调用了virtio_intr_init这个函数,传入的barid是1，表明是写在1号bar的地方
	if (virtio_interrupt_init(&blk->base, virtio_uses_msix())) {
		/* call close only for valid bctxt */
		if (!blk->dummy_bctxt)
			blockif_close(blk->bc);
		free(blk);
		return -1;
	}
	// 设置0号bar是io bar
	virtio_set_io_bar(&blk->base, 0);

	/*
	 * Register ops for virtio-blk Rescan
	 */
	if (register_vm_monitor_blkrescan == false) {

		register_vm_monitor_blkrescan = true;
		if (monitor_register_vm_ops(&virtio_blk_rescan_ops, ctx,
						"virtio_blk_rescan") < 0)
			pr_err("Rescan registration to VM monitor failed\n");
	}

	return 0;
}
```
#### virtio_blk_done
```rust
static void virtio_blk_done(struct blockif_req *br, int err)
{
	struct virtio_blk_ioreq *io = br->param;
	struct virtio_blk *blk = io->blk;

	if (err)
		DPRINTF(("virtio_blk: done with error = %d\n\r", err));

	/* convert errno into a virtio block error return */
	if (err == EOPNOTSUPP || err == ENOSYS)
		*io->status = VIRTIO_BLK_S_UNSUPP;
	else if (err != 0)
		*io->status = VIRTIO_BLK_S_IOERR;
	else
		*io->status = VIRTIO_BLK_S_OK;

	/*
	 * Return the descriptor back to the host.
	 * We wrote 1 byte (our status) to host.
	 */
	pthread_mutex_lock(&blk->mtx);
	vq_relchain(&blk->vq, io->idx, 1);
	// 最终会通过ioctl给guest写msix中断
	vq_endchains(&blk->vq, !vq_has_descs(&blk->vq));
	pthread_mutex_unlock(&blk->mtx);
}
```
#### virtio_linkup
```rust
/**
 * @brief Link a virtio_base to its constants, the virtio device,
 * and the PCI emulation.
 *
 * @param base Pointer to struct virtio_base.
 * @param vops Pointer to struct virtio_ops.
 * @param pci_virtio_dev Pointer to instance of certain virtio device.
 * @param dev Pointer to struct pci_vdev which emulates a PCI device.
 * @param queues Pointer to struct virtio_vq_info, normally an array.
 */
void
virtio_linkup(struct virtio_base *base, struct virtio_ops *vops,
	      void *pci_virtio_dev, struct pci_vdev *dev,
	      struct virtio_vq_info *queues,
	      int backend_type)
{
	int i;

	/* base and pci_virtio_dev addresses must match */
	// 因为base是传入参数blk（类型为virtio_blk）的第一个变量
	if ((void *)base != pci_virtio_dev) {
		pr_err("virtio_base and pci_virtio_dev addresses don't match!\n");
		return;
	}
	base->vops = vops;
	base->dev = dev;
	dev->arg = base;
	base->backend_type = backend_type;
	base->queues = queues;
	for (i = 0; i < vops->nvq; i++) {
		queues[i].base = base;
		queues[i].num = i;
	}
}
```
#### virtio_intr_init
```rust
/**
 * @brief Initialize MSI-X vector capabilities if we're to use MSI-X,
 * or MSI capabilities if not.
 *
 * We assume we want one MSI-X vector per queue, here, plus one
 * for the config vec.
 *
 *
 * @param base Pointer to struct virtio_base.
 * @param barnum Which BAR[0..5] to use.
 * @param use_msix If using MSI-X.
 *
 * @return 0 on success and non-zero on fail.
 */
int
virtio_intr_init(struct virtio_base *base, int barnum, int use_msix)
{
	int nvec;

	if (use_msix) {
		base->flags |= VIRTIO_USE_MSIX;
		VIRTIO_BASE_LOCK(base);
		// 清空vq，把msix_cfg_idx设置为VIRTIO_MSI_NO_VECTOR
		virtio_reset_dev(base); /* set all vectors to NO_VECTOR */
		VIRTIO_BASE_UNLOCK(base);
		// msix数量等于vq+1(vq是为了读写通知用，还有一个是为了config)
		nvec = base->vops->nvq + 1;
		// 添加msix cap
		if (pci_emul_add_msixcap(base->dev, nvec, barnum))
			return -1;
	} else
		base->flags &= ~VIRTIO_USE_MSIX;

	/* Only 1 MSI vector for acrn-dm */
	// 为什么初始化msi???不是msix吗
	pci_emul_add_msicap(base->dev, 1);

	/* Legacy interrupts are mandatory for virtio devices */
	pci_lintr_request(base->dev);

	return 0;
}
```
##### pci_emul_add_msixcap
```rust
int pci_emul_add_msixcap(struct pci_vdev *dev, int msgnum, int barnum)
{
	uint32_t tab_size;
	struct msixcap msixcap;	// 这个就是pci configuration space里的msix cap

	if (msgnum > MAX_MSIX_TABLE_ENTRIES) {
		pr_err("%s: Too many entries!\n", __func__);
		return -1;
	}

	tab_size = msgnum * MSIX_TABLE_ENTRY_SIZE;

	/* Align table size to nearest 4K */
	tab_size = roundup2(tab_size, 4096);
	// 说明table和pba在哪里去找
	dev->msix.table_bar = barnum;
	dev->msix.pba_bar   = barnum;
	dev->msix.table_offset = 0;
	dev->msix.table_count = msgnum;
	dev->msix.pba_offset = tab_size;
	dev->msix.pba_size = PBA_SIZE(msgnum);
	// 分配table内存，这是给我们持有的pci_vdev结构体分配的，真实的bar还不知道在哪里？？？然后初始化各个条目的vector_control为PCIM_MSIX_VCTRL_MASK
	if (pci_msix_table_init(dev, msgnum) != 0)
		return -1;
	// 初始化msixcap的相关信息（除了next cap以外的所有）
	pci_populate_msixcap(&msixcap, msgnum, barnum, tab_size);

	/* allocate memory for MSI-X Table and PBA */
	pci_emul_alloc_bar(dev, barnum, PCIBAR_MEM32,
				tab_size + dev->msix.pba_size);
	// 这一步主要是把msix这个加到pci cap链表里面
	return (pci_emul_add_capability(dev, (u_char *)&msixcap,
					sizeof(msixcap)));
}
```
###### pci_emul_alloc_pbar
```rust

int
pci_emul_alloc_pbar(struct pci_vdev *pdi, int idx, uint64_t hostbase,
		    enum pcibar_type type, uint64_t size)
{
	int error;
	uint64_t *baseptr = NULL, limit = 0, addr, mask, lobits, bar;
	struct io_rsvd_rgn *region;

	if ((size & (size - 1)) != 0)
		size = 1UL << flsl(size);	/* round up to a power of 2 */
	// 做一些bar idx是否合法的验证
	/* Enforce minimum BAR sizes required by the PCI standard */
	if (type == PCIBAR_IO) {
		if (size < 4)
			size = 4;
	} else {
		if (size < 16)
			size = 16;
	}

	if (idx > PCI_ROMBAR) {
		pr_err("%s: invalid bar number %d for PCI bar type\n", __func__, idx);
		return -1;
	}
	if (idx == PCI_ROMBAR) {
		/*
		 * It needs to pass the PCIBAR_ROM for PCI_ROMBAR idx. But as it
		 * is allocated from PCI_EMUL_MEM32 type, the internal type is
		 * changed to PCIBAR_MEM32
		 */
		if (type != PCIBAR_ROM) {
			pr_err("%s: invalid bar type %d for PCI ROM\n",
				__func__, type);
			return -1;
		}
		type = PCIBAR_MEM32;
	}
	switch (type) {
	case PCIBAR_NONE:
		baseptr = NULL;
		addr = mask = lobits = 0;
		break;
	case PCIBAR_IO:
		baseptr = &pci_emul_iobase;
		limit = PCI_EMUL_IOLIMIT;
		mask = PCIM_BAR_IO_BASE;
		lobits = PCIM_BAR_IO_SPACE;
		break;
	case PCIBAR_MEM64:
		if (idx + 1 > PCI_BARMAX) {
			pr_err("%s: invalid bar number %d for MEM64 type\n", __func__, idx);
			return -1;
		}
		/*
		 * FIXME
		 * Some drivers do not work well if the 64-bit BAR is allocated
		 * above 4GB. Allow for this by allocating small requests under
		 * 4GB unless then allocation size is larger than some arbitrary
		 * number (32MB currently). If guest booted by ovmf, then skip the
		 * workaround.
		 */
		if (!skip_pci_mem64bar_workaround && (size <= 32 * 1024 * 1024)) {
			baseptr = &pci_emul_membase32;
			limit = PCI_EMUL_MEMLIMIT32;
			mask = PCIM_BAR_MEM_BASE;
			lobits = PCIM_BAR_MEM_SPACE | PCIM_BAR_MEM_64;
			break;
		}

		/*
		 * XXX special case for device requiring peer-peer DMA
		 */
		if (size == 0x100000000UL)
			baseptr = &hostbase;
		else
			baseptr = &pci_emul_membase64;
		limit = PCI_EMUL_MEMLIMIT64;
		mask = PCIM_BAR_MEM_BASE;
		lobits = PCIM_BAR_MEM_SPACE | PCIM_BAR_MEM_64 |
			PCIM_BAR_MEM_PREFETCH;
		break;
	case PCIBAR_MEM32:
		// 进行一些参数设置,pci_emul_membase32还有iobase之类是在init_pci一开始利用ctx进行设置的
		baseptr = &pci_emul_membase32;
		limit = PCI_EMUL_MEMLIMIT32;
		mask = PCIM_BAR_MEM_BASE;
		lobits = PCIM_BAR_MEM_SPACE | PCIM_BAR_MEM_32;
		break;
	default:
		pr_err("%s: invalid bar type %d\n", __func__, type);
		return -1;
	}
	
	// 获取当前这个pdi对应idx的bar区域
	region = get_io_rsvd_rgn_by_vdev_idx(pdi, idx);
	if(region)
		addr = region->start;
	// 如果没分配的话就分配一个bar吧
	if (baseptr != NULL && !region) {
		error = pci_emul_alloc_resource(baseptr, limit, size, &addr, type);
		if (error != 0)
			return error;
	}

	pdi->bar[idx].type = type;
	pdi->bar[idx].addr = addr;
	pdi->bar[idx].size = size;

	if (idx == PCI_ROMBAR) {
		mask = PCIM_BIOS_ADDR_MASK;
		bar = addr & mask;
		/* enable flag will be configured later */
		pci_set_cfgdata32(pdi, PCIR_BIOS, bar);
	} else {
		/* Initialize the BAR register in config space */
		bar = (addr & mask) | lobits;
		pci_set_cfgdata32(pdi, PCIR_BAR(idx), bar);

		if (type == PCIBAR_MEM64) {
			pdi->bar[idx + 1].type = PCIBAR_MEMHI64;
			pci_set_cfgdata32(pdi, PCIR_BAR(idx + 1), bar >> 32);
		}
	}

	error = register_bar(pdi, idx); //注册bar，在这个里面会写当访问到这块mmio区域该调用哪个handler，此处注册的是pci_emul_mem_handler，因为类型是mem_32

	if(error != 0){
		/* FIXME: Currently, only gvt needs reserve regions.
		 * because gvt isn't firstly initialized, previous pci
		 * devices' bars may conflict with gvt bars.
		 * Use register_bar to detect this case,
		 * but this case rarely happen.
		 * If this case always happens, we need to
		 * change core.c code to ensure gvt firstly initialzed
		 */
		printf("%s failed to register_bar\n", pdi->name);
		return error;
	}

	return 0;
}
```
####### get_io_rsvd_rgn_by_vdev_idx
```rust
static struct io_rsvd_rgn *
get_io_rsvd_rgn_by_vdev_idx(struct pci_vdev *pdi, int idx)
{
	int i;
	// REGION_NUMS是32，为什么？？
	for(i = 0; i < REGION_NUMS; i++){
		if(reserved_bar_regions[i].vdev &&
			reserved_bar_regions[i].idx == idx &&
			reserved_bar_regions[i].vdev == pdi)
			return &reserved_bar_regions[i];
	}

	return NULL;
}
```

### pci_emul_mem_handler
```rust
// 都是调用vdev_barwrite和vdev_barread
static int pci_emul_mem_handler(struct vmctx *ctx, int vcpu, int dir, uint64_t addr, int size, uint64_t *val, void *arg1, long arg2)
{
	struct pci_vdev *pdi = arg1;
	struct pci_vdev_ops *ops = pdi->dev_ops;
	uint64_t offset;
	int bidx = (int) arg2;

	if (addr + size > pdi->bar[bidx].addr + pdi->bar[bidx].size) {
		pr_err("%s, Out of emulated memory range\n", __func__);
		return -ESRCH;
	}

	offset = addr - pdi->bar[bidx].addr;

	if (dir == MEM_F_WRITE) {
		if (size == 8) {
			(*ops->vdev_barwrite)(ctx, vcpu, pdi, bidx, offset,
					   4, *val & 0xffffffff);
			(*ops->vdev_barwrite)(ctx, vcpu, pdi, bidx, offset + 4,
					   4, *val >> 32);
		} else {
			(*ops->vdev_barwrite)(ctx, vcpu, pdi, bidx, offset,
					   size, bar_value(size, *val));
		}
	} else {
		if (size == 8) {
			uint64_t val_lo, val_hi;

			val_lo = (*ops->vdev_barread)(ctx, vcpu, pdi, bidx,
			                              offset, 4);
			val_lo = bar_value(4, val_lo);

			val_hi = (*ops->vdev_barread)(ctx, vcpu, pdi, bidx,
			                              offset + 4, 4);

			*val = val_lo | (val_hi << 32);
		} else {
			*val = (*ops->vdev_barread)(ctx, vcpu, pdi, bidx,
			                            offset, size);
			*val = bar_value(size, *val);
		}
	}

	return 0;
}
```
### virtio_pci_read & write
```rust

/**
 * @brief Handle PCI configuration space reads.
 *
 * Handle virtio standard register reads, and dispatch other reads to
 * actual virtio device driver.
 *
 * @param ctx Pointer to struct vmctx representing VM context.
 * @param vcpu VCPU ID.
 * @param dev Pointer to struct pci_vdev which emulates a PCI device.
 * @param baridx Which BAR[0..5] to use.
 * @param offset Register offset in bytes within a BAR region.
 * @param size Access range in bytes.
 *
 * @return register value.
 */
uint64_t
virtio_pci_read(struct vmctx *ctx, int vcpu, struct pci_vdev *dev,
		int baridx, uint64_t offset, int size)
{
	struct virtio_base *base = dev->arg;

	if (base->flags & VIRTIO_USE_MSIX) {
		if (baridx == pci_msix_table_bar(dev) ||
		    baridx == pci_msix_pba_bar(dev)) {
			return pci_emul_msix_tread(dev, offset, size);
		}
	}

	if (baridx == base->legacy_pio_bar_idx)
		return virtio_pci_legacy_read(ctx, vcpu, dev, baridx,
			offset, size);

	if (baridx == base->modern_mmio_bar_idx)
		return virtio_pci_modern_mmio_read(ctx, vcpu, dev, baridx,
			offset, size);

	if (baridx == base->modern_pio_bar_idx)
		return virtio_pci_modern_pio_read(ctx, vcpu, dev, baridx,
			offset, size);

	pr_err("%s: read unexpected baridx %d\r\n",
		base->vops->name, baridx);
	return size == 1 ? 0xff : size == 2 ? 0xffff : 0xffffffff;
}

/**
 * @brief Handle PCI configuration space writes.
 *
 * Handle virtio standard register writes, and dispatch other writes to
 * actual virtio device driver.
 *
 * @param ctx Pointer to struct vmctx representing VM context.
 * @param vcpu VCPU ID.
 * @param dev Pointer to struct pci_vdev which emulates a PCI device.
 * @param baridx Which BAR[0..5] to use.
 * @param offset Register offset in bytes within a BAR region.
 * @param size Access range in bytes.
 * @param value Data value to be written into register.
 */
void
virtio_pci_write(struct vmctx *ctx, int vcpu, struct pci_vdev *dev,
		 int baridx, uint64_t offset, int size, uint64_t value)
{
	struct virtio_base *base = dev->arg;

	if (base->flags & VIRTIO_USE_MSIX) {
		if (baridx == pci_msix_table_bar(dev) ||
		    baridx == pci_msix_pba_bar(dev)) {
			pci_emul_msix_twrite(dev, offset, size, value);
			return;
		}
	}

	if (baridx == base->legacy_pio_bar_idx) {
		virtio_pci_legacy_write(ctx, vcpu, dev, baridx,
			offset, size, value);
		return;
	}

	if (baridx == base->modern_mmio_bar_idx) {
		virtio_pci_modern_mmio_write(ctx, vcpu, dev, baridx,
			offset, size, value);
		return;
	}

	if (baridx == base->modern_pio_bar_idx) {
		virtio_pci_modern_pio_write(ctx, vcpu, dev, baridx,
			offset, size, value);
		return;
	}

	pr_err("%s: write unexpected baridx %d\r\n",
		base->vops->name, baridx);
}
```

## 结构体
### virtio_blk
```rust
struct virtio_blk {
	struct virtio_base base;
	pthread_mutex_t mtx;
	struct virtio_vq_info vq;
	struct virtio_blk_config cfg;
	bool dummy_bctxt; // Used in blockrescan. Indicate if the bctxt can be used 
	struct blockif_ctxt *bc;
	char ident[VIRTIO_BLK_BLK_ID_BYTES + 1];
	struct virtio_blk_ioreq ios[VIRTIO_BLK_RINGSZ];
	uint8_t original_wce;
};
```
#### virtio_base
``` rust
// 应该是virtio device公共都有的部分
struct virtio_base {
	struct virtio_ops *vops;	/**< virtio operations */
	int	flags;			/**< VIRTIO_* flags from above */
	bool	iothread;
	pthread_mutex_t *mtx;		/**< POSIX mutex, if any */
	struct pci_vdev *dev;		/**< PCI device instance */
	uint64_t negotiated_caps;	/**< negotiated capabilities */
	uint64_t device_caps;		/**< device capabilities */
	struct virtio_vq_info *queues;	/**< one per nvq */
	int	curq;			/**< current queue */
	uint8_t	status;			/**< value from last status write */
	uint8_t	isr;			/**< ISR flags, if not MSI-X */
	uint16_t msix_cfg_idx;		/**< MSI-X vector for config event */
	uint32_t legacy_pio_bar_idx;	/**< index of legacy pio bar */
	uint32_t modern_pio_bar_idx;	/**< index of modern pio bar */
	uint32_t modern_mmio_bar_idx;	/**< index of modern mmio bar */
	uint8_t config_generation;	/**< configuration generation */
	uint32_t device_feature_select;	/**< current selected device feature */
	uint32_t driver_feature_select;	/**< current selected guest feature */
	int cfg_coff;			/**< PCI cfg access capability offset */
	int backend_type;               /**< VBSU, VBSK or VHOST */
	struct acrn_timer polling_timer; /**< timer for polling mode */
	int polling_in_progress;        /**< The polling status */
};
```
##### pci_vdev
```rust
struct pci_vdev {
	struct pci_vdev_ops *dev_ops;
	struct vmctx *vmctx;
	uint8_t	bus, slot, func;
	char	name[PI_NAMESZ];
	int	bar_getsize;
	int	prevcap;
	int	capend;

	struct {
		int8_t	pin;
		enum lintr_stat	state;
		int		pirq_pin;
		int		ioapic_irq;
		pthread_mutex_t	lock;
	} lintr;

	struct {
		int		enabled;
		uint64_t	addr;
		uint64_t	msg_data;
		int		maxmsgnum;
	} msi;

	struct {
		int	enabled;
		int	table_bar;
		int	pba_bar;
		uint32_t table_offset;
		int	table_count;
		uint32_t pba_offset;
		int	pba_size;
		int	function_mask;
		struct msix_table_entry *table;	/* allocated at runtime */
		void	*pba_page;
		int	pba_page_offset;
	} msix;

	void	*arg;		/* devemu-private data */

	uint8_t	cfgdata[PCI_REGMAX + 1];	//这个就是configuration space的内容
	/* 0..5 is used for PCI MMIO/IO bar. 6 is used for PCI ROMbar */
	struct pcibar bar[PCI_BARMAX + 2];
};
```
###### pci_vdev_ops
```rust
struct pci_vdev_ops {
	char	*class_name;		/* Name of device class */

	/* instance creation */
	int	(*vdev_init)(struct vmctx *, struct pci_vdev *,
			     char *opts);

	/* instance deinit */
	void	(*vdev_deinit)(struct vmctx *, struct pci_vdev *,
			char *opts);

	/* ACPI DSDT enumeration */
	void	(*vdev_write_dsdt)(struct pci_vdev *);

	/* ops related to physical resources */
	void	(*vdev_phys_access)(struct vmctx *ctx, struct pci_vdev *dev);

	/* config space read/write callbacks */
	int	(*vdev_cfgwrite)(struct vmctx *ctx, int vcpu,
			       struct pci_vdev *pi, int offset,
			       int bytes, uint32_t val);
	int	(*vdev_cfgread)(struct vmctx *ctx, int vcpu,
			      struct pci_vdev *pi, int offset,
			      int bytes, uint32_t *retval);

	/* BAR read/write callbacks */
	void	(*vdev_barwrite)(struct vmctx *ctx, int vcpu,
				 struct pci_vdev *pi, int baridx,
				 uint64_t offset, int size, uint64_t value);
	uint64_t  (*vdev_barread)(struct vmctx *ctx, int vcpu,
				struct pci_vdev *pi, int baridx,
				uint64_t offset, int size);
};
```
###### msix_table_entry
```rust
// 这个结构定义和hypervisor文件夹下是一样的
struct msix_table_entry {
	uint64_t	addr;
	uint32_t	msg_data;
	uint32_t	vector_control;
} __attribute__((packed));
```
##### virtio_blk_ops
```rust
static struct virtio_ops virtio_blk_ops = {
	"virtio_blk",		// our name 
	1,			// we support 1 virtqueue 
	sizeof(struct virtio_blk_config), // config reg size 
	virtio_blk_reset,	// reset
	virtio_blk_notify,	// device-wide qnotify 
	virtio_blk_cfgread,	// read PCI config 
	virtio_blk_cfgwrite,	// write PCI config 
	NULL,			// apply negotiated features 
	NULL,			// called on guest set status
};
```
#### blockif_ctxt
```rust
struct blockif_ctxt {
	int			fd;
	int			isblk;
	int			candiscard;
	int			rdonly;
	off_t			size;
	int			sub_file_assign;
	off_t			sub_file_start_lba;
	struct flock		fl;
	int			sectsz;
	int			psectsz;
	int			psectoff;
	int			max_discard_sectors;
	int			max_discard_seg;
	int			discard_sector_alignment;
	int			closing;
	pthread_t		btid[BLOCKIF_NUMTHR];
	pthread_mutex_t		mtx;
	pthread_cond_t		cond;

	// Request elements and free/pending/busy queues
	TAILQ_HEAD(, blockif_elem) freeq;
	TAILQ_HEAD(, blockif_elem) pendq;
	TAILQ_HEAD(, blockif_elem) busyq;
	struct blockif_elem	reqs[BLOCKIF_MAXREQ];

	// write cache enable
	uint8_t			wce;
};
```
#### virtio_blk_ioreq
```rust
struct virtio_blk_ioreq {
	struct blockif_req req;
	struct virtio_blk *blk;
	uint8_t *status;
	uint16_t idx;
};
```
### msixcap
```rust
struct msixcap {
	uint8_t		capid;
	uint8_t		nextptr;
	uint16_t	msgctrl;
	uint32_t	table_info;	/* bar index and offset within it */
	uint32_t	pba_info;	/* bar index and offset within it */
} __attribute__((packed));
```
### io_rsvd_rgn
```rust
struct io_rsvd_rgn {
	uint64_t start;
	uint64_t end;
	int idx;
	int bar_type;
	/* if vdev=NULL, it also indicates this io_rsvd_rgn is not used */
	struct pci_vdev *vdev;
};
```

# hypervisor
- init_pcpu_comm_post
  - init_guest_mode
    - launch_vms
      - prepare_vm
        - create_vm
          - init_vpci
            - vpci_init_vdevs
            - register_mmio_emulation_handler
            - register_pio_emulation_handler: 
              1. Intercept and handle I/O ports CF8h
              2. Intercept and handle I/O ports CFCh -- CFFh 