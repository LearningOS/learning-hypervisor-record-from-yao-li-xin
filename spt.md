

### 第一阶段地址翻译 (Stage-1: GVA → GPA)



**页表项定义** :

```rust
//s1pt.rs:25-40
bitflags::bitflags! {
    pub struct DescriptorAttr: usize {
        const V = 1 << 0;        // Valid - 有效位
        const D = 1 << 1;        // Dirty - 脏位
        const PLV = 0b11 << 2;   // Privilege Level - 特权级
        const MAT = 0b11 << 4;   // Memory Access Type - 内存访问类型
        const G = 1 << 6;        // Global - 全局位
        const P = 1 << 7;        // Present - 存在位
        const W = 1 << 8;        // Writable - 可写位
        const NR = 1 << 61;      // Not Readable - 不可读
        const NX = 1 << 62;      // Not Executable - 不可执行
        const RPLV = 1 << 63;    // Relative Privilege Level Check
    }
}
```

**页表项结构** :

```rust
#[derive(Clone, Copy, Debug)]
#[repr(transparent)]
pub struct PageTableEntry(usize);

const PTE_PPN_MASK: usize = 0x0000_ffff_ffff_f000; // 12-47位
```

**页表激活** :

```rust
impl PagingInstr for S1PTInstr {
    unsafe fn activate(root_pa: HostPhysAddr) {
        // 设置页表基址寄存器
        pgdh::set_base(root_pa);
        pgdl::set_base(root_pa);
        // 设置TLB缺失处理程序
        tlbrentry::set_tlbrentry(tlb_refill_handler as usize);
    }
}
```

### 第二阶段地址翻译 (Stage-2: GPA → HPA)



**页表项定义** :

```rust
bitflags::bitflags! {
    pub struct DescriptorAttr: usize {
        const V = 1 << 0;        // Valid
        const D = 1 << 1;        // Dirty
        
        // 特权级定义
        const PLV = 0b11 << 2;   // Privilege Level Range
        const PLV0 = 0b00 << 2;  // PLV0
        const PLV1 = 0b01 << 2;  // PLV1
        const PLV2 = 0b10 << 2;  // PLV2
        const PLV3 = 0b11 << 2;  // PLV3
        
        // 内存访问类型
        const MAT = 0b11 << 4;   // Memory Access Type Range
        const MAT_SUC = 0b00 << 4; // Strongly-Ordered Uncached
        const MAT_CC = 0b01 << 4;  // Coherent Cached
        const MAT_WB = 0b10 << 4;  // Weakly-Ordered Uncached
        
        const G = 1 << 6;        // Global
        const P = 1 << 7;        // Present
        const W = 1 << 8;        // Writable
        const NR = 1 << 61;      // Not Readable
        const NX = 1 << 62;      // Not Executable
        const RPLV = 1 << 63;    // Relative Privilege Level Check
    }
}
```



**页表激活**:
```rust
impl PagingInstr for S2PTInstr {
    unsafe fn activate(root_pa: HostPhysAddr) {
        // 设置页表配置
        super::paging::set_pwcl_pwch_stlbps();
        // 设置页表基址寄存器
        pgdh::set_base(root_pa);
        pgdl::set_base(root_pa);
        // 设置TLB缺失处理程序
        tlbrentry::set_tlbrentry(tlb_refill_handler as usize);
    }
}
```

### TLB管理

#### 1. TLB充填

```rust
#[no_mangle]
#[naked]
extern "C" fn tlb_refill_handler() {
    unsafe {
        asm!(
        "csrwr      $r12, {LOONGARCH_CSR_TLBRSAVE}",  // 保存寄存器
        "csrrd      $r12, {LOONGARCH_CSR_PGD}",       // 读取页表基址
        "lddir      $r12, $r12, 3",                   // 加载第4级页表
        "ori        $r12, $r12, {PAGE_WALK_MASK}",
        "xori       $r12, $r12, {PAGE_WALK_MASK}",
        "lddir      $r12, $r12, 2",                   // 加载第3级页表
        "ori        $r12, $r12, {PAGE_WALK_MASK}",
        "xori       $r12, $r12, {PAGE_WALK_MASK}",
        "lddir      $r12, $r12, 1",                   // 加载第2级页表
        "ori        $r12, $r12, {PAGE_WALK_MASK}",
        "xori       $r12, $r12, {PAGE_WALK_MASK}",
        "ldpte      $r12, 0",                         // 加载页表项0
        "ldpte      $r12, 1",                         // 加载页表项1
        "tlbfill",                                    // 填充TLB
        "csrrd      $r12, {LOONGARCH_CSR_TLBRSAVE}",  // 恢复寄存器
        "ertn",                                       // 异常返回
        options(noreturn)
        );
    }
}
```

#### 2. TLB刷新

```rust
    fn flush_tlb(vaddr: Option<Self::VirtAddr>) {
        unsafe {
            if let Some(vaddr) = vaddr {
                // 刷新特定地址的TLB条目
                core::arch::asm!("dbar 0; invtlb 0x05, $r0, {}", in(reg) vaddr.as_usize());
            } else {
                // 刷新整个TLB
                core::arch::asm!("dbar 0; invtlb 0, $r0, $r0");
            }
        }
    }
```

TLB相关寄存器

```rust
pub gcsr_tlbrentry: usize, // TLBRENTRY - TLB缺失处理程序入口
pub gcsr_tlbrbadv: usize,  // TLBRBADV - TLB缺失地址
pub gcsr_tlbrera: usize,   // TLBRERA - TLB缺失返回地址
pub gcsr_tlbrsave: usize,  // TLBRSAVE - TLB缺失保存寄存器
pub gcsr_tlbrelo0: usize,  // TLBRELO0 - TLB条目低32位0
pub gcsr_tlbrelo1: usize,  // TLBRELO1 - TLB条目低32位1
pub gcsr_tlbrehi: usize,   // TLBREHI - TLB条目高32位
pub gcsr_tlbrprmd: usize,  // TLBRPRMD - TLB缺失特权模式
```

### 页表配置

`paging.rs:620-637`

```rust
pub fn set_pwcl_pwch_stlbps() {
    set_dir3_base(12 + 9 + 9 + 9);  // 第4级页表基址位
    set_dir3_width(9);              // 第4级页表宽度
    set_dir2_base(12 + 9 + 9);      // 第3级页表基址位
    set_dir2_width(9);              // 第3级页表宽度
    set_dir1_base(12 + 9);          // 第2级页表基址位
    set_dir1_width(9);              // 第2级页表宽度
    set_ptbase(12);                 // 第1级页表基址位
    set_ptwidth(9);                 // 第1级页表宽度
    set_pte_width(8);               // 页表项宽度(8字节)
    stlbps::set_ps(12);             // 页大小(4KB)
}
```



```rust
// 页表索引计算
const fn p4_index(vaddr: usize) -> usize {
    (vaddr >> (12 + 27)) & (ENTRY_COUNT - 1)  // 第4级索引
}

const fn p3_index(vaddr: usize) -> usize {
    (vaddr >> (12 + 18)) & (ENTRY_COUNT - 1)  // 第3级索引
}

const fn p2_index(vaddr: usize) -> usize {
    (vaddr >> (12 + 9)) & (ENTRY_COUNT - 1)   // 第2级索引
}

const fn p1_index(vaddr: usize) -> usize {
    (vaddr >> 12) & (ENTRY_COUNT - 1)         // 第1级索引
}
```



1. 属性编码模式：

- AArch64：使用MAIR寄存器索引 + 复杂位字段组合

- 龙芯：直接编码 + 简化位字段

1. 权限控制模式：

- AArch64：S2AP_RO/S2AP_WO + XN + NS

- 龙芯：PLV + W + NR + NX + RPLV

1. TLB管理模式：

- AArch64：使用TLBI指令 + 内存屏障

- 龙芯：使用INVTLB指令

1. 内存类型模式：

- AArch64：Device/Normal/NormalNonCache

- 龙芯：StronglyUncached/CoherentCached/WeaklyUncached





``` rust
// 1. 属性定义
bitflags::bitflags! { pub struct DescriptorAttr: u64 { ... } }

// 2. 页表项结构
#[repr(transparent)] pub struct A64PTEHV(u64);

// 3. 实现GenericPTE
impl GenericPTE for A64PTEHV { ... }

// 4. 分页元数据
pub struct A64HVPagingMetaData;
impl PagingMetaData for A64HVPagingMetaData { ... }

// 5. 类型别名
pub type NestedPageTable<H> = PageTable64<A64HVPagingMetaData, A64PTEHV, H>;
```



```rust
const MAT_SUC = 0b00 << 4;  // Strongly-Ordered Uncached - 强序非缓存
const MAT_CC = 0b01 << 4;   // Coherent Cached - 一致性缓存
const MAT_WB = 0b10 << 4;   // Weakly-Ordered Uncached - 弱序非缓存
```

