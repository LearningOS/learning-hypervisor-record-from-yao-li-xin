# learning-hypervisor-record-from-yao-li-xin

## 一, Learning axvisor

### Day1

1. 今天主要阅读axvisor架构设计手册, 中断部分没有读完.
   通过宏,实现相关trait, 并拓展task字段

   了解了API设计: 在链接阶段完成接口与实现的绑定, 查看了相关源码.

   通过 trait 关联的方式绑定vm与pagehandle.

2. 阅读了vcpus.rs的源码, 了解了AxVcpu, VmxVcpu等的数据结构, 以及启动主cpu, 辅cpu 及其相关联函数等代码


接下来阅读源码应该根据启动流程依次阅读, 并继续完成架构设计学习的笔记.

