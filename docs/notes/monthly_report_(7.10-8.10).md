## 月度工作进展报告（7.10 ~ 8.10）已移动到 `docs/notes/月报(7.10 - 8.10).md`

------

### 一、工作总结

本月的主要工作围绕 **虚拟化文档阅读与整理**、**启动流程分析**、**测试代码开发与优化** 以及 **龙芯架构虚拟化环境准备**

#### 1. 虚拟化学习

##### LA虚拟化配置

- 参考 **hvisor** 与LA文档，初步整理了 LoongArch 虚拟化启动配置。
- 整理了相关文档: [hvisor_note](https://github.com/manchangfengxu/learning-hypervisor-record-from-yao-li-xin/blob/main/hvisor_note.md)

##### axvisor架构,流程

- 阅读了 **Axvisor 架构设计手册**
- 阅读axvisor分析 `vcpus.rs`、`main.rs` 等核心模块，了解相关数据结构及启动流程。
- 整理了相关文档: [axvisor_start](https://github.com/manchangfengxu/learning-hypervisor-record-from-yao-li-xin/blob/main/axvisor start.md)

##### hypervisor搭建流程学习

- 完成 h 系列实验并撰写了实验笔记
- 相关文档: [riscv_hypervisor](https://github.com/manchangfengxu/learning-hypervisor-record-from-yao-li-xin/blob/main/riscv_hypervisor.md)

#### 2. 测试 与 文档 的PR贡献

##### 	测试代码

- 为 `axaddrspace/src/frame.rs` 编写了单元测试,并进行了多轮重构与优化：

  - 相关 PR 已合并：[feat: Implement unit tests for PhysFrame by manchangfengxu · Pull Request #17 · arceos-hypervisor/axaddrspace](https://github.com/arceos-hypervisor/axaddrspace/pull/17)

##### 文档

-  `arceos-org/page_table_multiarch` 文档部分
-  相关 PR 已合并: [fix(docs): Clarify alignment behavior and fix typo in map function by manchangfengxu · Pull Request #23 · arceos-org/page_table_multiarch](https://github.com/arceos-org/page_table_multiarch/pull/23)

#### 3.[日志仓库](https://github.com/LearningOS/learning-hypervisor-record-from-yao-li-xin?tab=readme-ov-file#day24)



------

### 二、当前进度与下月计划

#### 当前进度

- **Axvisor 启动流程** 与 **LoongArch 虚拟化准备** 已完成初步文档整理。
- **测试代码** 和 **文档** 相关贡献已合并主线。
- 协助参与 `axplat_crates` 中龙芯,`boot.rs`下虚拟化启动部分的添加。

#### 下月计划

1. 与助教及同学继续细化 **axvisor for LA64** 的任务拆分，形成更具体的开发计划。
2. 通过小组会议确定接口设计方案
3. 推进 `loongarch64_vcpu` 仓库的接口与数据结构实现。
4. 深入学习 `oscomp_arceos_tour_u` 系列实验，进一步掌握 LoongArch64 架构特性与底层实现细节。

------

