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
- **** 为常量、静态变量、`MockHal` 结构体及其方法增加了全面的文档注释 (`///`)，极大地提升了代码的可理解性。
- 添加注释阐释了 `MockHal` 上 `Debug` derive 的目的







