# learning-hypervisor-record-from-yao-li-xin

## 一, Learning axvisor

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
- 更细粒度的阅读了一下源码, 因为大部分都被分离成了crate, 所以源码基本都是和启动相关的, 进行了更细粒度的整理, 在axvisor源码.md里
- 读完了vm-exit的设计, 了解了axvcpu, axvm, axvmm的多层vm_exit处理机制







