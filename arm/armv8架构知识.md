


- [前言](#前言)
- [aarch64](#aarch64)
    - [寄存器：](#寄存器)
      - [cache功能项：](#cache功能项)
    - [原子性操作：](#原子性操作)
    - [中断机制](#中断机制)
      - [`Cortex-A9`：](#cortex-a9)
      - [`Cortex-A53`：](#cortex-a53)
    - [MMU](#mmu)
      - [基础知识点](#基础知识点)
        - [页表结构](#页表结构)
        - [CPU访问流程](#cpu访问流程)
    - [数据读写流向](#数据读写流向)
- [总结](#总结)

---

# 前言


本文主要描述armv8架构

---

# aarch64 
### 寄存器：

 - 通用寄存器(general-purpose registers)：
有R0~R30 共计31个通用寄存器,x30通常作为LR寄存器
 <div align="center">
<img src="https://i-blog.csdnimg.cn/blog_migrate/b1fc04393ed25e121a5504a41e57c73d.png" width="70%">
<p>图片1.通用寄存器</p>
</div>

 - 栈指针寄存器(`SP`)
 一个64bit栈指针寄存器

 <div align="center">
<img src="https://i-blog.csdnimg.cn/blog_migrate/22a8f354567453697d4261345a73f72b.png" width="70%">
<p>图片2.SP寄存器</p>
</div>

 - 程序计数寄存器(`PC`)
 一个64bit的程序计数寄存器

 <div align="center">
<img src="https://i-blog.csdnimg.cn/blog_migrate/3281131d2b7af2b5df012b3f6fcbd82f.png" width="70%">
<p>图片3.PC寄存器</p>
</div>

 - SIMD&FP寄存器(`SIMD&FP`)
 32个128bit的SIMD&FP操作寄存器


 <div align="center">
<img src="https://img-blog.csdnimg.cn/direct/a1139af5508c4c1197f26e89d2dd9218.png " width="70%">
<img src="https://img-blog.csdnimg.cn/direct/b54b83d74eef453a9268ec7dc1b15567.png " width="70%">
<p>图片4.SIMD&FP寄存器</p>
</div>

```bash
uint64_t * pinValue = 0x1000;
uint64_t * poutValue = 0x2000;
__asm__ volatile(
"ldnp q0,q1,[%[ValueIn1]]\n"
"stnp q0,q1,[%[ValueOut1]]\n"
:
:[ValueIn1]"r"(pinValue),[ValueOut1]"r"(poutValue)
:"memory","x0"
);
__asm__ volatile(
"ld4 {v0.2d,v1.2d,v2.2d,v3.2d},[%[ValueIn1]]\n"
"st4 {v0.2d,v1.2d,v2.2d,v3.2d},[%[ValueOut1]]\n"
:
:[ValueIn1]"r"(pinValue),[ValueOut1]"r"(poutValue)
:"memory","x0"
);
```

- FPCR, FPSR寄存器(`FPCR, FPSR`)
 2个64bit的SIMD&FP运算状态寄存器，存储SIMD指令及浮点运算过程中的状态

- 可伸缩矢量寄存器(`SVE`)
 32个128bit的SIMD&FP操作寄存器，用于SIMD指令及浮点运算


 <div align="center">
<img src="https://img-blog.csdnimg.cn/direct/767821fddc7348a9aa4f57cff8481d01.png " width="70%">
<p>图片4.SIMD&FP寄存器</p>
</div>

#### cache功能项：
 - Cache type寄存器(`Cache Type Register`)
主要功能如下：
	- 任何受指令缓存维护指令影响的指令缓存的最小行长度。
	- 任何受数据缓存维护指令影响的数据或统一缓存的最小行长度。
	- 一级指令缓存的缓存索引和标记策略。

 <div align="center">
<img src="https://i-blog.csdnimg.cn/direct/91be07b25c4d4e509eed8ffa8d6fd367.png" width="100%">
<img src="https://i-blog.csdnimg.cn/direct/649cf80d20ba44e8953ad1b49f197a99.png" width="100%">
<p>图片5.cache type 寄存器</p>
</div>

### 原子性操作：

 - 对齐的LDR、STR为single-copy atomic
   - 单次加载/存储操作不可分割
   - 支持最大64bit数据宽度（8字节）
   - 适用于：指针操作、基本数据类型访问
   - 示例：`LDR X0, [X1]` / `STR X0, [X1]`

 - 对齐的LDNP、STNP为2次single-copy atomic
   - 每次操作实际执行两个独立的8字节访问
   - 支持128bit数据宽度（16字节）
   - 不保证两个8字节操作的原子顺序
   - 适用于：流式数据操作、非关键数据块拷贝
   - 示例：`LDNP Q0, Q1, [X2]` / `STNP Q0, Q1, [X3]`

### 中断机制

```c
ADD R0, R1, R2    ; 假设执行这条指令时触发中断
MOV R3, #1        ; 下一条指令
```
在Cortex-A53/A9中，当执行ADD指令时发生中断：

对ADD指令的处理有**两种**可能：

1.如果ADD已经进入执行阶段(Execute stage)，则会执行完成
2.如果ADD还在译码或更早的流水线阶段，则可能会被取消(canceled)并重新执行


关于**LR**值的设置：

如果ADD指令被完整执行了，**LR**会指向MOV指令的地址
如果ADD指令被取消了，**LR**会指向ADD指令的地址



这种行为设计的原因是：

保证指令的原子性
确保中断返回后能够正确恢复执行
维护处理器状态的一致性

#### `Cortex-A9`：


采用经典的ARMv7-A架构
支持精确中断机制
使用6-8级流水线
**当中断发生时**：

1.完成当前指令执行
2.LR = PC - 4（对于IRQ/FIQ，指向MOV）(ps:若未完成时,LR指向ADD指令)
3.支持虚拟化扩展（可选）

#### `Cortex-A53`：
采用ARMv8-A架构
同样支持精确中断
使用8级流水线
提供了更多的异常等级（EL0-EL3）
**当中断发生时**：

1.完成当前指令执行
2.LR = PC - 4（在AArch32模式下）
3.在AArch64模式下，使用ELR_ELx寄存器保存返回地址
4.支持更复杂的异常处理机制
### MMU

#### 基础知识点

##### 页表结构
Linux 通常采用四级页表结构:

- PGD (Page Global Directory)
- PUD (Page Upper Directory)
- PMD (Page Middle Directory)
- PTE (Page Table Entry)



在**ARM64**上:

- TTBR0_EL1: 指向用户空间页表基地址
- TTBR1_EL1: 指向内核空间页表基地址
- 每个进程都有自己的用户空间页表(TTBR0)
- 所有进程共享同一个内核页表(TTBR1)


```c
页表切换过程
switch_mm() {
    // 1. 获取新进程的页表基地址
    pgd = next->mm->pgd;
    
    // 2. 切换TTBR0寄存器
    write_ttbr0(pgd);
    
    // 3. 刷新TLB
    flush_tlb_local();  // 清除旧进程的TLB项
    
    // 4. 更新ASID(Address Space ID)
    if (prev_asid != next_asid) {
        update_asid(next_asid);
    }
}
```
内存管理的关键概念：

- 每个进程有独立的虚拟地址空间
- 内核空间在所有进程中映射相同
- 使用ASID来避免切换时完全刷新TLB
- 页表项包含访问权限、缓存属性等标志位



##### CPU访问流程

**CPU取指令阶段:**


CPU首先获取程序计数器(PC)中的虚拟地址
这个虚拟地址需要被转换为物理地址才能访问实际的内存


地址转换过程:

- 首先检查TLB(Translation Lookaside Buffer):

	CPU会先查找TLB缓存
	TLB条目包含ASID(Address Space ID)和虚拟地址到物理地址的映射
	ASID用于区分不同进程的地址空间,避免TLB刷新
	如果找到匹配的TLB条目(TLB hit),直接使用缓存的物理地址

-  TLB未命中时的页表遍历:

	CPU读取TTBR(Translation Table Base Register)获取页表基地址
	对于ARM架构通常是两级页表结构
	使用虚拟地址的不同位段索引页表
	最终得到物理页框号(PFN)


**Cache访问:**


得到物理地址后,先检查Cache
- 现代CPU通常采用VIPT(Virtually Indexed Physically Tagged)Cache
- Cache使用虚拟地址的低位作为索引,物理地址作为Tag
- 如果Cache命中,直接返回数据
- Cache缺失则需要访问内存


**实际执行过程中的优化:**


- CPU会进行指令预取,填充指令Cache
- 分支预测会提前计算可能用到的地址
- MMU包含micro-TLB专门用于指令地址转换
- Cache和TLB并行查询以提高性能

**这个过程涉及几个关键组件的协同工作:**

- ASID确保每个进程有独立的地址空间视图
- TTBR指向当前进程的页表结构
- TLB缓存最近使用的地址转换结果
- Cache系统隐藏内存访问延迟

**当进程切换时:**

- 需要更新TTBR指向新进程的页表
- 可能需要刷新TLB(除非使用ASID)
- Context Switch会影响Cache性能

**这些机制共同保证了虚拟内存的正确性和性能。每个组件都针对特定问题提供优化:**

- ASID解决进程隔离问题
- TLB解决地址转换速度问题
- Cache解决内存访问延迟问题


MMU中可配置cache属性。某块RAM配置了cacheable属性，若存放了指令，则会使用Icache，若触发了访问，则会使用Dcache。


### 数据读写流向

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/247a0351540f41ab93eab6830334a461.png)

数据流向说明：

- CPU执行STR指令：

 	- 数据首先被写入Write Buffer
	- CPU可以继续执行后续指令，不需等待


-	Write Buffer到Cache：

	- Write Buffer会异步地将数据写入L1 Cache
	- 写入时机由硬件控制
	- 如果遇到DMB指令，必须等待Write Buffer清空


Cache层级之间：

- 在Write-Back策略下：

	- 数据从L1 Cache写回L2 Cache
数据从L2 Cache写回RAM


写回时机：

- Cache line被替换
- 显式的Clean操作
- Cache一致性请求





Write Buffer特点：

- 纯硬件实现，对软件透明
- 有限的条目数（通常4-8个）
- 支持合并写操作（Write Merging）
- 可以被内存屏障指令控制

使用Write Buffer的好处：

- 提高写操作性能
- 支持写合并，减少对内存的访问
- 隐藏内存访问延迟

在不同场景下的行为：

普通写操作：

```c
STR R1, [R0]    ; 数据进入Write Buffer
                ; CPU继续执行
```



带内存屏障的写操作：
```c
STR R1, [R0]    ; 数据进入Write Buffer
DMB             ; 等待Write Buffer清空到Cache
```


带Cache清理的写操作：
```c
STR R1, [R0]    ; 数据进入Write Buffer
DSB             ; 等待Write Buffer清空到Cache
Clean Cache     ; 数据写回到RAM
```

对程序员的影响：

- 普通数据访问无需关心Write Buffer
- 需要同步时必须使用内存屏障
- 要写入RAM时需要Cache维护操作
# 总结
