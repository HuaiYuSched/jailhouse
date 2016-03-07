jailhouse analysis about vt-d

###Config file 
The config file predefine the system's config and the data in this file will be load to defined memory region which will be assigned to some struct point like system_config as we discuss later. We can hold config file configs/qemu-vm.c file as a example in this doc. Now let's take a glance on the struct on this file.   
```
struct {
	struct jailhouse_system header;
	__u64 cpus[1];
	struct jailhouse_memory mem_regions[13];
	struct jailhouse_irqchip irqchips[1];
	__u8 pio_bitmap[0x2000];
	struct jailhouse_pci_device pci_devices[8];
	struct jailhouse_pci_capability pci_caps[5];
} __attribute__((packed)) config
```
This struct and it's content in config.c file will be compiled to binary format. This binary can be load to the memory before the jailhouse hypervisor running by cmd jailhouse enable.  
##Jailhouse Memory Layout 
An overall (physical) layout of Jailhouse memory region is shown at Fig. 1:   

    +-----------+-------------------+-----------------+------------+--------------+------+
    | Guest RAM | hypervisor_header | Hypervisor Code | cpu_data[] | system_confg | Heap |
    +-----------+-------------------+-----------------+------------+--------------+------+
                |__start                              |__page_pool                       |hypervisor_header.size

Fig. 1 Jailhouse physical memory layout scheme. Lowercase labels are global symbols or common function argument names. Reference to the article in LWN.   

##Init_early 
After entry, the init_early is the first function get invoked.   
Firstly, it will assign system_config to the virtual address.   
```
system_config = (struct jailhouse_system *)
		(JAILHOUSE_BASE + core_and_percpu_size);
```		
As we can see, the content of the system_config comes from qemu_vm.c:  
```
.header = {
		.signature = JAILHOUSE_SYSTEM_SIGNATURE,
		.hypervisor_memory = {
			.phys_start = 0x3b000000,
			.size = 0x600000,
		},
		.debug_console = {
			.phys_start = 0x3f8,
		},
		.platform_info.x86 = {
			.mmconfig_base = 0xb0000000,
			.mmconfig_end_bus = 0xff,
			.pm_timer_address = 0x608,
			.iommu_base = {
				0xfed90000,
			},
		},
		.interrupt_limit = 256,
		.root_cell = {
			.name = "QEMU-VM",

			.cpu_set_size = sizeof(config.cpus),
			.num_memory_regions = ARRAY_SIZE(config.mem_regions),
			.num_irqchips = ARRAY_SIZE(config.irqchips),
			.pio_bitmap_size = ARRAY_SIZE(config.pio_bitmap),
			.num_pci_devices = ARRAY_SIZE(config.pci_devices),
			.num_pci_caps = ARRAY_SIZE(config.pci_caps),
		},
	},
```

Then paging_init function is invoked. In this function, jailhouse will initialize the mem_pool and remap_pool.   

##Cell_init  
This function will initial Cell in jailhouse. Not only for the root_cell, every time you create cell, this function will be invoked.   
After return from paging_init, Jailhouse will assign the root_cell as the value in config file we put before. Then the cell_init will calculate the value of struct cell, alloc mmio pages for the in mmio_cell_init for cell mmio manage struct.  

##arch_init_early
In this function, it will initialize the apic and idt.   

##cpu_init 
This function will read the cpu register and store it to struct cpu_data   

##init_late->iommu_init 
//TODO how to describe this. firstly, find the small n of interrput_limit.  

```
	for (n = 0; (1UL << n) < (system_config->interrupt_limit); n++)
```

Then alloc pages as int_remap_table from mem_pool for every interrput. Count how many iommu the cell has and alloc pages from remap_pool and mem_pool as dmar_reg_base and unit_inv_queue respectively. For each iommu region, the iommu_init will map the page alloced from remap_pool to iommu_base address by function paging_create. After that, we can read the mmio_regist from this address. The Following process about register is highly related to the Vt-d specification.  
 1.check whether this device support Vt-d or not.   
 2. get the cache type of this device   
 3. get the sagaw and page cache mode.    
 4. get the interrput mode from extend capability register.   
 5. get the control register and set interrupt and queue invalidation and dma remapping as enable.   
 6. vtd_init_ir_emulation will fill the struct root_cell_units. This function will initialize the interrrupt remapping support.   

 ```
	/*
	 * Derive vdt_paging from very similar x86_64_paging,
	 * replicating 0..3 for 4 levels and 1..3 for 3 levels.
	 */
	memcpy(vtd_paging, &x86_64_paging[4 - dmar_pt_levels],
	       sizeof(struct paging) * dmar_pt_levels);
	for (n = 0; n < dmar_pt_levels; n++)
		vtd_paging[n].set_next_pt = vtd_set_next_pt;
	if (!(sllps_caps & VTD_CAP_SLLPS1G))
		vtd_paging[dmar_pt_levels - 3].page_size = 0;
	if (!(sllps_caps & VTD_CAP_SLLPS2M))
		vtd_paging[dmar_pt_levels - 2].page_size = 0;
```
Do not understand this code very clearly 
###iommu_cell_init
This function will fill the cell struct and reserve interrupt remap region for the interrupt of the cell.

##map_root_memory_regions  

##pci_init pci_cell_init iommu_add_pci_device 
In this function, the system read the mmcfg information from the system_config. Then map a free page to the address of mmcfg.(Where is the address of mmio_base in this function derives from?)
Then in the pci_cell_init,  the jailhouse invoke mmio_region_register function with function pci_mmconfig_access_handler. Tring to access to this region mmio will cause vm-exit, which will be handle in vmm by the function now you register. After that, the jailhouse find all the pci_device belongs to cell and use pci_add_physical_device to add this pci device to cell and register msix region handler. Eventually, iommu_add_pci_device is invoked to perform this add action. iommu_add_pci_device do this two things:   

 1. get the bdf and the according to this bdf, find the context_entry of this device. If this root_entry is not present, enable it and alloc a page for context_entry_table. For this part, refer to vt-d specification.  
 2. vtd_reserve_int_remap_region  
