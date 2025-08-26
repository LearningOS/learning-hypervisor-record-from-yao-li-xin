# learning-hypervisor-record-from-yao-li-xin

## 目录

### 7月
- [Day1](#day1)
- [Day2](#day2)
- [Day3](#day3)
- [Day4](#day4)
- [Day5](#day5)
- [Day6](#day6)
- [Day7](#day7)
- [Day8](#day8)
- [Day9](#day9)
- [Day10](#day10)
- [Day11](#day11)
- [Day12](#day12)
- [Day13](#day13)
- [Day14](#day14)
- [Day15](#day15)
- [Day16](#day16)
- [Day17](#day17)
- [Day18](#day18)
- [Day19](#day19)
- [Day20](#day20)
- [Day21](#day21)
- [Day22](#day22)
- [Day23](#day23)
- [Day24](#day24)
- [Day25](#day25)
- [Day26](#day26)
- [Day27](#day27)
- [Day28](#day28)
- [Day29](#day29)
- [Day30](#day30)
- [Day31](#day31)
- [Day32](#day32)
- [Day33](#day33)
- [Day34](#day24)



### Day1

1. 今天主要阅读axvisor架构设计手册, 中断部分没有读完.

    - 通过宏,实现相关trait, 并拓展task字段

    - 了解了API设计: 在链接阶段完成接口与实现的绑定, 查看了相关源码.

    - 通过 trait 关联的方式绑定vm与pagehandle.

2. 阅读了vcpus.rs的源码, 了解了AxVcpu, VmxVcpu等的数据结构, 以及启动主cpu, 辅cpu 及其相关联函数等代码

接下来阅读源码应该根据启动流程依次阅读, 并继续完成架构设计学习的笔记.

### Day2

1. 架构文档方面
    - 继续阅读了虚拟中断, 了解了hypervisor对物理中断的截获, 以及重新路由等概念, GICV, GICH的作用等, 但对具体的寄存器操作没有去了解
    - GICv3的内容有很多,全部详细了解的话感觉太花费时间,后面开始快速过了一遍,了解了中断配置和处理流程ITS的工作流程,以及对应的部分直通的虚拟化来解决纯软件模拟复杂的问题. 
    - 读完了设备部分, PIC设备, buffer链,以及相应的可用/已用环

2. 阅读了main.rs相关的使能, 启动代码,

    - 了解到#[percpu::def_percpu], 为每核心分配不同实例的过程宏

    - hardware_enable()主要是和硬件交互的代码, 没有全部了解.
    - 主要流程: id, 时钟, 硬件寄存器等的配置 -> 为gust vm配置主vcpu -> 设置vm为running-> 将vcpu放置到运行队列等待调度

### Day3
- 更细粒度的阅读了一下源码, 因为大部分都被分离成了crate, 所以源码基本都是和启动相关的, 进行了更细粒度的整理, 在`axvisor start.md`里
- 读完了vm-exit的设计, 了解了axvcpu, axvm, axvmm的多层vm_exit处理机制

### Day4

1. 查看了关于编写rust文档的官方文档, 同时编写了一些demo了解如何组织文档的结构

    - **层级对应代码结构：** 文档结构严格遵循模块（文件夹和 `mod.rs`/`.rs` 文件）的嵌套关系。
    - **`lib.rs` 或 `main.rs` 的 `//!`：** 用于整个Crate的首页概览。
    - **`mod.rs` 或模块对应的 `.rs` 文件的 `//!`：** 用于该模块的概览页面。
    - **所有公共（`pub`）项前的 `///`：** 生成具体函数/结构体等的详细文档。
    - **Markdown：** 文档注释内用Markdown格式化文本、链接（`[`Item`]` 内部链接）和代码块。

    - **特殊标题：** 如 `# Examples`、`# Panics` 等，帮助组织内容。

2. 了解了编写测试代码的规范

3. 了解了发布流程

4. 查看了苏助教给的一些pr示例, 了解了结构优化方面)能做哪些内容, 大抵总结了这些内容:

    - 对于重复, 简单的struct建立及相关implement, 可以用宏来代替
    - 使用`trait`关联来减少泛型链
    - 一些辅助: 增加trait表达, 边界处理等
5. 看了基于 ARMv8 两阶段地址翻译的内存虚拟化的视频

### Day5
在查看src/address_space/backend/alloc.rs时, 发现`map_alloc`使用的map存在一些文档问题
- 拼写错误

- 地址对齐的描述问题

- PR链接: [fix(docs): Clarify alignment behavior and fix typo in map function](https://github.com/arceos-org/page_table_multiarch/pull/23) 
  
### Day6

- 学习了ARMv8 下的 I/O 与中断虚拟化
- 尝试编写测试代码, 本地没什么问题, 准备再审视一下()

### Day7
- 测试代码已push到我fork仓库的分支上了, 等苏助教评审, 改进
- 对于Drop的多线程测试还没有解决
- [链接](https://github.com/manchangfengxu/axaddrspace/tree/feat/add-frame-tests)

### Day8
今天完善了测试代码, 并解决了多线程环境下Drop测试的问题
- 通过引入一个全局的 TEST_MUTEX 来强制需要共享状态的测试串行执行，解决了并行测试中因资源竞争导致的随机失败和 Drop 计数不准确的问题。
- 为`test_alloc_dealloc_cycle()`和`test_alloc_no_memory()`增加alloc是否成功下的Drop的测试
- 增加了关键部分的注释
- PR链接: [feat: Implement unit tests for PhysFrame](https://github.com/arceos-hypervisor/axaddrspace/pull/17)



### Day9

今天主要是根据苏助教的 review 改善代码，要更改的细节有点多。

- 添加了 `assert_matches!`
- 引入了 `BASE_PADDR` 和 `MEMORY_LEN` 常量，来配置模拟内存。
- 将 `ALLOC_FAIL` 重命名为 `ALLOC_SHOULD_FAIL`。
- 实现了 `MockHal::reset_state()` 方法，确保在每个测试执行前所有共享静态变量都能被重置到干净状态
- `test_alloc_zero` 和 `test_fill_operation` 中的内存访问断言已重构，改用切片迭代器 (`&[u8; PAGE_SIZE].iter().all()`)。
- 为常量、静态变量、`MockHal` 结构体及其方法增加了全面的文档注释 (`///`)，增加了更多的comment。
- 添加注释阐释了 `MockHal` 上 `Debug` derive 的目的



### Day10

1. 测试代码

   - 加入了多页面fill的测试

   - 改了一些注释

   - 更改了`assert_matches!`的验证方式

2. 参考x86vcpu实现, 了解虚拟化硬件配置, 阅读了`fn hardware_enable(&mut self) -> AxResult`的代码

   - 检查cpu型号是否支持虚拟化, CR4 VMXE检查是否已经开启
   - 启用XSAVE/XRSTOR
   - 配置 IA32_FEATURE_CONTROL MSR 并 锁定
   - CR0 和 CR4 寄存器检查(各个位是否符合)
   - 读取IA32_VMX_BASIC_MSR, 并验证信息
   - 给VMCS id和分配frame
   - 设置 CR4 的 VMXE 位
   - 执行 `VMXON`

   
   
### Day11
尝试合并陈宏和我的测试代码依赖的Mock分配器, 实现AxMmHal和PagingHandler, 迁移到全局共享的地方

### Day12
- commit链接: [feat: Implement unit tests for PhysFrame](https://github.com/arceos-hypervisor/axaddrspace/pull/17/commits/46e9581d51327f6e121774f7ef9fd8f142b4e4ac)
- 重构了`test_fill_multiple_frames()`, 更自动化

### Day13
今天回家, 晚上读了一些龙芯vcpu架构的相关文档, 但是没有找到有关虚拟化初始硬件配置与检测的内容.

### Day14
1. 更新了测试代码, 用了苏助教给的axin crate, 已经push到我fork仓库的分支上了
2. 了解了虚拟机的的启动和退出流程, 查看了相关源码

### Day15
- 官网没有找到与虚拟化拓展相关的内容, 尝试向工作人员要一下
- 了解了axvisor中vcpu设计框架, 相关`struct`, `trait`
- [PR已合并](https://github.com/arceos-hypervisor/axaddrspace/pull/17)

### Day16
- 跑通最新的arceos for loongarch, 跑通了example下不需要` loongarch64-linux-musl-gcc`的几个测例
- 想从LVZ文档中找到虚拟化使能相关信息, 只找到了判断是否支持虚拟化拓展, 询问了谢助教得到一些建议.

### Day17
- 运行axvisor, 跟踪路径, 并阅读源码 
- 回顾了一下hvisor的论文

### Day18
根据hvisor项目, 初步了解了龙芯虚拟化启动的准备

### Day19
总结了hvisor中, 虚拟化环境开启需要的前置条件, 在[hvisor_note](hvisor_note.md)中, 并对照架构手册进行相关寄存器查阅

### Day20
完成了两个实验, 了解了最简hypervisor, 以及如何进行二阶段地址映射, 初步写了文档, 没有整理完.

### Day21
查阅了axplat_crates中的boot代码了解了龙芯架构,非虚拟化的启动流程, 并整理了相关文档:[boot](boot.md)

### Day22
与ch, xzj讨论了如何去完成axvisor for LA的启动部分boot.rs的虚拟化配置添加
* 在系统启动初期，首先进行 CPU 能力检测。通过检查 `CPUCFG` 寄存器的 `LVZ` 扩展位（`bit10`），来确认当前硬件是否支持虚拟化功能。

* 在硬件支持确认后，进行 Hypervisor 自身的初始化工作。配置必要的控制寄存器，例如 `GTLBC` 等，为管理 Guest 虚拟机和虚拟化资源做好准备。

*  `struct VMContext` ，存储虚拟机状态（包括 CPU 寄存器、内存配置、中断状态等），以便在虚拟机进出时进行保存和恢复。

* CPU 虚拟化实现： 补充实现 CPU 虚拟化相关的逻辑。这部分工作将专注于 Guest 寄存器的管理、特权指令的截获与模拟，以及 Guest 与 Host 之间的模式切换。

* 内存虚拟化实现： 借鉴其他架构,都是**两级地址翻译**。同时，关注 **TLB（地址转换后备缓冲区）同步**问题，确保 Guest 和 Host 的 TLB 一致性。`axcpu` 模块中已提供的相关接口。



### Day23
完成了两个实验的整理, 在[hypervisor.md](hypervisor.md)里, 闪到腰了, 卧床休息几天

### Day24
完成了h系列所有实验, 并对4个实验笔记进行了整理[hypervisor](riscv_hypervisor.md);

### Day25
- 查阅了axcpu中有关龙芯架构的部分, 初步整理了一些硬件的接口与封装的实现.
- 通过x86_vcpu, 寻找其相关的虚拟化硬件实现代码的位置, 以检测虚拟化支持为例: 最终发现在raw-cpuid crate标准库里
```rust
    pub fn cpuid_count(a: u32, c: u32) -> CpuIdResult {
        // Safety: CPUID is supported on all x86_64 CPUs and all x86 CPUs with
        // SSE, but not by SGX.
        let result = unsafe { self::arch::__cpuid_count(a, c) };

        CpuIdResult {
            eax: result.eax,
            ebx: result.ebx,
            ecx: result.ecx,
            edx: result.edx,
        }
    }
```

### Day26
初步确立了分工, 根据ch总结的资料并结合项目源码对vcpu设计有了更进一步的理解

### Day27
通过hvisor, 初步整理了两阶段地址翻译, 同时继续阅读axvisor整体设计.

### Day28
- 整理了两阶段地址翻译的信息(仍需完善)[spt.md](./spt.md)
- 初步尝试适配`axaddrspace`

### Day29
- 初步完成`axaddrspace`中`npt/arch`的适配
    - 实现了 GenericPTE 和 PagingMetaData trait
    - 实现了与 MappingFlags 的双向转换
    - 使用 bitflags 定义页表项属性
    - 实现了架构特定的 TLB 刷新
    - 支持了不同内存属性的访问
- 开始尝试和谢助教完成vcpu的适配

### Day30
重新查阅手册, 修复了`axaddrspace`的相关错位
- 位错误
- 名称缩写转向手册

### Day31
- 完成了vcpu整体框架的搭建, 利用拓展写出了一份todo列表
- 实现了架构无关的代码

### Day32
- 尝试了vcpu中架构相关的搭建, 寄存器的读取, tlb刷新等
- 对`axaddrspace`中的一些编译错误进行解决

### Day33
- 解决了`axaddrspace`中的编译错误,以及补充了`flush_tlb` 上传到了xzj的仓库中, 已提pr
- 初步根据xzj的仓库配置axvisor, 遇到一些问题

### Day34
- 尝试运行`axvisor`, 更改相关配置文件引用本地crate, 解决了嵌套页表的问题
- 解决了新出现的问题, 在配置文件中增加了变量
- 目前问题在于axvm, 及其相关联crate的适配, 准备晚上初步讨论
- 目前新分支已提到xzj的仓库中
   





