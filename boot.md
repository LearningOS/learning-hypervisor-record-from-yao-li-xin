

```12:15:axplat_crates/platforms/axplat-loongarch64-qemu-virt/src/boot.rs
#[unsafe(link_section = ".bss.stack")]
static mut BOOT_STACK: [u8; BOOT_STACK_SIZE] = [0; BOOT_STACK_SIZE];

#[unsafe(link_section = ".data")]
static mut BOOT_PT_L0: Aligned4K<[LA64PTE; 512]> = Aligned4K::new([LA64PTE::empty(); 512]);

#[unsafe(link_section = ".data")]
static mut BOOT_PT_L1: Aligned4K<[LA64PTE; 512]> = Aligned4K::new([LA64PTE::empty(); 512]);
```

- `BOOT_STACK`: 启动栈，大小为 `BOOT_STACK_SIZE`
- `BOOT_PT_L0`: 页表第0级（L0），包含512个页表项
- `BOOT_PT_L1`: 页表第1级（L1），包含512个页表项

### 页表初始化

```17:32:axplat_crates/platforms/axplat-loongarch64-qemu-virt/src/boot.rs
unsafe fn init_boot_page_table() {
    unsafe {
        let l1_va = va!(&raw const BOOT_PT_L1 as usize);
        // 0x0000_0000_0000 ~ 0x0080_0000_0000, table
        BOOT_PT_L0[0] = LA64PTE::new_table(axplat::mem::virt_to_phys(l1_va));
        // 0x0000_0000..0x4000_0000, VPWXGD, 1G block
        BOOT_PT_L1[0] = LA64PTE::new_page(
            pa!(0),
            MappingFlags::READ | MappingFlags::WRITE | MappingFlags::DEVICE,
            true,
        );
        // 0x8000_0000..0xc000_0000, VPWXGD, 1G block
        BOOT_PT_L1[2] = LA64PTE::new_page(
            pa!(0x8000_0000),
            MappingFlags::READ | MappingFlags::WRITE | MappingFlags::EXECUTE,
            true,
        );
    }
}
```

两级页表设置：

- L0页表指向L1页表
- L1页表映射了两个1GB的内存区域：
  - `0x0000_0000..0x4000_0000`: 设备内存区域（可读写，设备属性）
  - `0x8000_0000..0xc000_0000`: 普通内存区域（可读写执行）

### FP/SIMD启用函数

```34:42:axplat_crates/platforms/axplat-loongarch64-qemu-virt/src/boot.rs
fn enable_fp_simd() {
    // FP/SIMD needs to be enabled early, as the compiler may generate SIMD
    // instructions in the bootstrapping code to speed up the operations
    // like `memset` and `memcpy`.
    #[cfg(feature = "fp-simd")]
    {
        axcpu::asm::enable_fp();
        axcpu::asm::enable_lsx();
    }
}
```

早期启用浮点和SIMD指令，因为编译器可能在启动代码中生成这些指令来加速内存操作。

### MMU初始化函数

```44:49:axplat_crates/platforms/axplat-loongarch64-qemu-virt/src/boot.rs
fn init_mmu() {
    axcpu::init::init_mmu(
        axplat::mem::virt_to_phys(va!(&raw const BOOT_PT_L0 as usize)),
        PHYS_VIRT_OFFSET,
    );
}
```

初始化内存管理单元（MMU），设置页表基地址和物理-虚拟地址偏移。

###  主CPU启动入口点

```51:75:axplat_crates/platforms/axplat-loongarch64-qemu-virt/src/boot.rs
#[unsafe(naked)]
#[unsafe(no_mangle)]
#[unsafe(link_section = ".text.boot")]
unsafe extern "C" fn _start() -> ! {
    core::arch::naked_asm!("
        ori         $t0, $zero, 0x1     # CSR_DMW1_PLV0
        lu52i.d     $t0, $t0, -2048     # UC, PLV0, 0x8000 xxxx xxxx xxxx
        csrwr       $t0, 0x180          # LOONGARCH_CSR_DMWIN0
        ori         $t0, $zero, 0x11    # CSR_DMW1_MAT | CSR_DMW1_PLV0
        lu52i.d     $t0, $t0, -1792     # CA, PLV0, 0x9000 xxxx xxxx xxxx
        csrwr       $t0, $t0, 0x181          # LOONGARCH_CSR_DMWIN1

        # Setup Stack
        la.global   $sp, {boot_stack}
        li.d        $t0, {boot_stack_size}
        add.d       $sp, $sp, $t0       # setup boot stack

        # Init MMU
        bl          {enable_fp_simd}    # enable FP/SIMD instructions
        bl          {init_boot_page_table}
        bl          {init_mmu}          # setup boot page table and enable MMU

        csrrd       $a0, 0x20           # cpuid
        li.d        $a1, 0              # TODO: parse dtb
        la.global   $t0, {entry}
        jirl        $zero, $t0, 0",
        // ... 符号绑定
    )
}
```

主CPU的启动入口点, 包括：

1. 设置直接映射窗口（DMW）配置
2. 设置启动栈
3. 启用FP/SIMD
4. 初始化页表
5. 初始化MMU
6. 获取CPU ID
7. 跳转到主函数



