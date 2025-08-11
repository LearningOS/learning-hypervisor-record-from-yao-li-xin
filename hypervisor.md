## 最简Hypervisor

### 最简Hypervisor执行流程：

1. 加载Guest OS内核Image到新建地址空间。
2. 准备虚拟机环境，设置特殊上下文。
3. 结合特殊上下文和指令sret切换到V模式，即VM-ENTRY。
4. OS内核只有一条指令，调用sbi-call的关机操作。
5. 在虚拟机中，`sbi-call`超出V模式权限，导致VM-EXIT退出虚拟机，切换回Hypervisor。
6. Hypervisor响应VM-EXIT的函数检查退出原因和参数，进行处理，由于是请求关机，清理虚拟机后，退出。

### 需要做的：

1. 准备能够进入虚拟化模式的特殊上下文。

2. 为响应VM-EXIT设置处理函数。

   

   
### 进入虚拟化的准备
1. ISA寄存器设置: `misa`第7位代表Hypervisor扩展的启用/禁止, `OpenSBI`启动时显示ISA扩展支持

2. `hstatus设置`: 第7位SPV记录上一次进入HS特权级前的模式, 标记之前的"状态".执行sret时，根据SPV决定是返回用户态，还是返回虚拟化模式

3. Hypervisor首次启动Guest的内核之前，伪造上下文作准备：

   设置Guest的sstatus，让其初始特权级为Supervisor；

   设置Guest的sepc为OS启动入口地址VM_ENTRY，VM_ENTRY值就是内核启动的入口地址，

   对于Riscv64，其地址是**0x8020_0000**。

### vcpu上下文切换

每个vCPU维护一组上下文状态，分别对应Host和Guest。

从Hypervisor切断到虚拟机时，暂存Host中部分可能被破坏的状态；加载Guest状态；

然后执行sret完成切换。封装该过程的专门函数**run_guest**。

### VM-EXIT准备



## 地址空间

1. 简化实验模型:
    省略VM虚拟机这一级对象，只有虚拟地址空间(aspace)和VCpu(上下文状态)两个对象，
  - 虚拟地址空间: 负责定位虚拟机资源
  - VCpu: 负责运行状态的跟踪和切换

2. Guest与Host的地址空间关系

- **小循环**：在Guest内部完成，指的是Guest的操作系统（GuestOS）使用`satp`寄存器将Guest虚拟地址空间（GVA）映射到Guest物理地址空间（GPA）的过程 2222。

- **大循环**：需要Hypervisor参与，指的是Hypervisor使用`hgatp`寄存器将Guest物理地址空间（GPA）映射到Host物理地址空间（HPA），以补齐映射或响应请求的过程。


地址映射的流程是：

GVA–> (vsatp)->GPA–> (hgatp) ->HPA



### 虚拟模式切换

- **_run_guest**：从 Hypervisor 切换进入 Guest（V=1 S 态），完成“保存 Hypervisor→加载 Guest→sret 进入”的全过程。
- **_guest_exit**：Guest 发生异常/中断/系统调用等 VM-EXIT 返回 Hypervisor 后的入口，完成“保存 Guest→恢复 Hypervisor→返回 Rust 处理函数”的全过程。

#### _run_guest：进入 Guest 的关键步骤
```4:19:arceos/exercises/simple_hv/src/guest.S
_run_guest:
    /* Save hypervisor state */
    /* Save hypervisor GPRs ... */
    sd ra/gp/tp/s0...s11/sp, ({hyp_*})(a0)
```
- **保存 Hypervisor 通用寄存器**到 `VmCpuRegisters.hyp_regs`（`a0` 指向 vCPU 结构体）。避免 Guest 破坏宿主状态。

```32:46:arceos/exercises/simple_hv/src/guest.S
/* Swap in guest CSRs. */
ld t1, ({guest_sstatus})(a0); csrrw t1,sstatus,t1; sd t1,({hyp_sstatus})(a0)
ld t1, ({guest_hstatus})(a0); csrrw t1,hstatus,t1
ld t1, ({guest_scounteren})(a0); csrrw t1,scounteren,t1; sd t1,({hyp_scounteren})(a0)
ld t1, ({guest_sepc})(a0); csrw sepc,t1
```
- **换入 Guest 的关键 CSR**：
  - `sstatus`: 控制返回特权级（由预先设置的 `spp=Supervisor` 决定）。
  - `hstatus`: 控制进入 V=1（预先设置 `spv=Guest`，`spvp=Supervisor`）。
  - `scounteren`: 计数器访问控制。
  - `sepc`: Guest 返回地址（入口点）。

```47:55:arceos/exercises/simple_hv/src/guest.S
/* stvec 指向 _guest_exit，确保 Guest 退出后回到此处 */
la t1,_guest_exit; csrrw t1,stvec,t1; sd t1,({hyp_stvec})(a0)
/* sscratch 存 vCPU 指针，便于 _guest_exit 取回 */
csrrw t1, sscratch, a0; sd t1,({hyp_sscratch})(a0)
```
- **设置陷入返回地址 stvec** 到 `_guest_exit`。
- **用 sscratch 携带 vCPU 指针**（vCPU 结构体地址，供退出时恢复现场）。

```56:89:arceos/exercises/simple_hv/src/guest.S
/* 恢复 Guest GPRs 和 sp/a0，最终 sret 进入 Guest */
ld ra/gp/tp/s0...t6/sp/a0, ({guest_*})(a0)
sret
```
- **恢复 Guest 通用寄存器并 sret**：sret 从 `sepc` 返回到 Guest；因 `hstatus.spv` 及 `sstatus.spp` 预置，进入 VS 模式的 S 态。

#### _guest_exit：从 Guest 回 Hypervisor 的关键步骤
```91:100:arceos/exercises/simple_hv/src/guest.S
_guest_exit:
/* 取回 vCPU 指针：a0 <- sscratch；同时把 guest 的 a0 暂存进 sscratch */
csrrw a0, sscratch, a0
/* 保存 Guest GPRs 回 vCPU 结构体 */
sd ra/gp/tp/s0...s11/t0..t6/sp, ({guest_*})(a0)
```
- **取回 vCPU 指针**：`a0` 现在指向 vCPU 结构体；把 Guest 的 a0 临时塞到 `sscratch` 里。
- **保存 Guest 的通用寄存器**回 `guest_regs`，保留退出现场。

```128:130:arceos/exercises/simple_hv/src/guest.S
/* 把刚才暂存在 sscratch 的 Guest a0 存回结构体 */
csrr t0, sscratch
sd   t0, ({guest_a0})(a0)
```
- **保存 Guest 的 a0**：配合前面的 `csrrw`，把 Guest 的 a0 放回 `guest_regs`。

（在完整文件中，接下来会完成：交换/恢复 CSR 到 Hypervisor、保存 `sepc`、恢复 Hypervisor 的通用寄存器、`ret` 返回 Rust 层的 `vmexit_handler` 继续处理。）

4. 设备模拟和透传(默认)两种模式的实现

	- 模拟:用map_alloc申请页帧完成映射, 从file1中加载内容填充页帧
	- 用map_linear直接映射, qemu(宿主平台)的pflash#2的地址区域
	
	- 核心触发点：Guest 访问未映射设备区域（如 pflash#2）→ VM-EXIT: NestedPageFault → 在 Hypervisor 中按策略完成映射后继续执行。
	
	- pflash#2 物理地址（GPA）来自平台配置：
	```27:35:arceos/platforms/riscv64-qemu-virt.toml
	mmio-regions = [
	    ...
	    ["0x2200_0000", "0x0200_0000"],  # PFlash
	    ...
	]
	```
	
	1. 透传模式（Pass-through，默认）
	- 思路：把 Guest 访问的 GPA 线性映射到宿主侧同地址区域，直接透传给 QEMU 的设备。
	- 处理代码（在 NestedPageFault 分支内）：
	```61:71:arceos/tour/h_3_0/src/main.rs
	assert_eq!(addr, 0x2200_0000.into(), "Now we ONLY handle pflash#2.");
	let mapping_flags = MappingFlags::from_bits(0xf).unwrap();
	// Passthrough-Mode
	let _ = aspace.map_linear(addr, addr.as_usize().into(), 4096, mapping_flags);
	```
	- 原理实现：
	```103:129:arceos/modules/axmm/src/aspace.rs
	pub fn map_linear(&mut self, start_vaddr, start_paddr, size, flags) -> AxResult { ... }
	```
	```20:31:arceos/modules/axmm/src/backend/linear.rs
	pt.map_region(start, |va| PhysAddr::from(va - pa_va_offset), size, flags, ...)
	```
	- 适用：MMIO 设备可直接由宿主处理，开销低、简单。
	- 注意：地址与长度必须 4K 对齐；flags 按需设置（示例 0xf = RWXU），多页需按页循环映射。
	
	2. 模拟模式（Emulator）
	- 思路：不给设备透传，改为给到一块“自管”内存页，按需把文件数据写入，后续访问都落在这块内存，实现“假设备”。
	- 处理代码（同一位置，切换为模拟）：
	```65:71:arceos/tour/h_3_0/src/main.rs
	// Emulator-Mode
	let buf = "pfld";
	aspace.map_alloc(addr, 4096, mapping_flags, true);
	aspace.write(addr, buf.as_bytes());
	```
	- 原理实现：
	```140:159:arceos/modules/axmm/src/aspace.rs
	pub fn map_alloc(&mut self, start, size, flags, populate: bool) -> AxResult { ... }
	```
	```43:55:arceos/modules/axmm/src/backend/alloc.rs
	if populate { // 预分配页帧并映射，每页清零
	    pt.map(... flags).is_ok()
	} else {      // 懒分配：先占坑，缺页再分配
	    pt.map_region(... flags=empty)
	}
	```
	- 懒分配时的缺页处理钩子：
	```88:104:arceos/modules/axmm/src/backend/alloc.rs
	handle_page_fault_alloc(vaddr, orig_flags, pt, populate) { ... remap(vaddr, new_frame, orig_flags) ... }
	```
	- 适用：需要完全掌控设备行为（复现/录制/检测/修改数据），便于调试和教学。
	- 注意：populate=true 会立即分配物理页；否则会在第一次访问时触发页分配。文件加载可用 `aspace.write` 或先 `translated_byte_buffer` 再写入。






