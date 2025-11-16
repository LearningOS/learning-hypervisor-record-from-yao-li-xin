## 从hvisor中提取龙芯架构虚拟化环境开启

### 1. 虚拟化环境配置

#### **系统启动初始化** (`src/arch/loongarch64/entry.rs`)
```rust
// 配置DMW (Direct Mapping Window)
csrwr $r12, {LOONGARCH_CSR_DMW0}  // 0x8000000000000001
csrwr $r12, {LOONGARCH_CSR_DMW1}  // 0x9000000000000011

// 配置关键CSR寄存器
csrwr $r12, {LOONGARCH_CSR_CRMD}   // PLV=0, IE=0, PG=1
csrwr $r12, {LOONGARCH_CSR_PRMD}   // PLV=0, PIE=1, PWE=0
csrwr $r12, {LOONGARCH_CSR_EUEN}   // 禁用各种异常
```

#### **Trap向量安装** (`src/arch/loongarch64/trap.rs`)
```rust
pub fn install_trap_vector() {
    // 强制禁用中断
    crmd::set_ie(false);
    // 清除UEFI固件的定时器配置
    tcfg::set_en(false);
    ticlr::clear_timer_interrupt();
    
    // 设置CSR.EENTRY为_hyp_trap_vector
    ecfg::set_vs(0);
    eentry::set_eentry(_hyp_trap_vector as usize);
    
    // 启用浮点运算
    euen::set_fpe(true); // 基础浮点
    euen::set_sxe(true); // 128位SIMD
    euen::set_asxe(true); // 256位SIMD
    
    enable_global_interrupt()
}
```

#### **CPU和虚拟机上下文初始化** (`src/arch/loongarch64/cpu.rs`)
```rust
impl ArchCpu {
    pub fn new(cpuid: usize) -> Self {
        let ret = ArchCpu {
            ctx: super::trap::dump_reset_gcsrs(), // 获取默认GCSR值
            stack_top: 0,
            cpuid,
            power_on: false,
            init: false,
        };
        ret
    }
}
```

#### **内存虚拟化配置** (`src/arch/loongarch64/zone.rs`)
```rust
pub fn pt_init(&mut self, mem_regions: &[HvConfigMemoryRegion]) -> HvResult {
    for region in mem_regions {
        match region.mem_type {
            MEM_TYPE_RAM => {
                self.gpm.insert(MemoryRegion::new_with_offset_mapper(
                    region.virtual_start as GuestPhysAddr,
                    region.physical_start as HostPhysAddr,
                    region.size as _,
                    MemFlags::READ | MemFlags::WRITE | MemFlags::EXECUTE,
                ))?;
            }
            MEM_TYPE_IO => {
                // IO内存映射
            }
            MEM_TYPE_VIRTIO => {
                // VirtIO设备映射
            }
        }
    }
    Ok(())
}
```

#### **虚拟化硬件启用** (`src/arch/loongarch64/trap.rs`)
```rust
#[no_mangle]
pub fn _vcpu_return(ctx: usize) {
    let vm_id = zone_id + 1;
    
    // 1. 启用Guest模式
    gstat::set_gid(vm_id);           // 设置Guest ID
    gstat::set_pgm(true);            // 启用Guest模式
    
    // 2. 配置Guest TLB控制
    gtlbc::set_use_tgid(true);       // 启用TLB Guest ID
    gtlbc::set_tgid(vm_id);          // 设置TLB Guest ID
    
    // 3. 配置Guest控制寄存器
    gcfg::set_matc(0x1);             // 设置内存访问类型控制
    
    // 4. 禁用敏感特权资源异常
    gcfg::set_topi(false);           // 禁用特权指令异常
    gcfg::set_toti(false);           // 禁用特权定时器异常
    gcfg::set_toe(false);            // 禁用特权异常异常
    gcfg::set_top(false);            // 禁用特权页异常
    gcfg::set_tohu(false);           // 禁用特权硬件异常
    gcfg::set_toci(0x2);             // 设置特权控制接口
    
    // 5. 配置中断传递
    gintc::set_hwip(0xff);           // 允许所有硬件中断传递给Guest
    
    // 6. 启用中断
    prmd::set_pie(true);             // 启用中断
}
```

### 2. 进入虚拟化环境的方式

#### **CPU运行流程** (`src/arch/loongarch64/cpu.rs`)
```rust
pub fn run(&mut self) -> ! {
    // 激活Guest页表
    this_cpu_data().activate_gpm();
    self.power_on = true;
    
    // 初始化寄存器
    for i in 0..32 {
        self.ctx.x[i] = 0;
    }
    self.ctx.gcsr_cpuid = 0;
    
    // 刷新TLB
    unsafe {
        asm!("invtlb 0, $r0, $r0");
    }
    
    // 调用硬件启用函数
    super::trap::_vcpu_return(ctx_addr as usize);
}
```

#### **通过 `ertn` 指令进入Guest模式**
在 `_hyp_trap_return` 函数的最后：
```rust
asm!(
    // 恢复所有寄存器状态
    "ld.d $r0, $r3, 0",
    // ... 恢复其他寄存器
    "csrwr $r3, {LOONGARCH_CSR_DESAVE}",
    "ertn",  // 关键指令：进入Guest模式
    LOONGARCH_CSR_DESAVE = const 0x502
);
```

### 3. 退出虚拟化环境的方式

#### **异常处理退出** (`src/arch/loongarch64/trap.rs`)
```rust
#[no_mangle]
#[naked]
#[link_section = ".trap_entry"]
extern "C" fn _hyp_trap_vector() {
    // 保存所有Guest状态到ZoneContext
    asm!(
        "csrwr $r3, {LOONGARCH_CSR_DESAVE}",
        "csrrd $r3, {LOONGARCH_CSR_SAVE3}",
        // 保存32个通用寄存器
        "addi.d $r3, $r3, -768",
        "st.d $r0, $r3, 0",
        // ... 保存其他寄存器
        // 保存所有GCSR寄存器
        "gcsrrd $r12, {LOONGARCH_GCSR_CRMD}",
        "st.d $r12, $r3, 256+8*1",
        // ... 保存其他GCSR
        "bl trap_handler",
    );
}
```

#### **Trap处理器** (`src/arch/loongarch64/trap.rs`)
```rust
pub fn trap_handler(mut ctx: &mut ZoneContext) {
    let estat_ = estat::read();
    let ecode = estat_.ecode();
    let esubcode = estat_.esubcode();
    
    // 处理不同类型的异常
    handle_exception(ecode, esubcode, era_.raw(), is, badi_.inst() as usize, badv_.vaddr(), ctx);
    
    // 恢复定时器配置
    write_gcsr_tcfg(ctx.gcsr_tcfg);
    write_gcsr_estat(ctx.gcsr_estat);
    
    // 返回Guest模式
    unsafe {
        let _ctx_ptr = ctx as *mut ZoneContext;
        _vcpu_return(_ctx_ptr as usize);
    }
}
```

### 4. 关键数据结构

#### **ZoneContext结构** (`src/arch/loongarch64/zone.rs`)
```rust
#[repr(C)]
#[repr(align(16))]
#[derive(Debug, Copy, Clone)]
pub struct LoongArch64ZoneContext {
    pub x: [usize; 32],              // 32个通用寄存器
    pub sepc: usize,                  // 异常返回地址
    
    // 通用控制和状态寄存器
    pub gcsr_crmd: usize,            // CRMD
    pub gcsr_prmd: usize,            // PRMD
    pub gcsr_euen: usize,            // EUEN
    // ... 其他GCSR寄存器
    
    // TLB寄存器
    pub gcsr_tlbidx: usize,          // TLBIDX
    pub gcsr_tlbehi: usize,          // TLBEHI
    // ... 其他TLB寄存器
    
    // 页表寄存器
    pub gcsr_pgdl: usize,            // PGDL
    pub gcsr_pgdh: usize,            // PGDH
    // ... 其他页表寄存器
}
```
