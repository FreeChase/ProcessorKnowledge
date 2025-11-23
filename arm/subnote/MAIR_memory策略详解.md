
## 🚀 AArch64 下 MAIR (Memory Attribute Indirection Register) 配置详解

**MAIR 的核心作用：** 在 AArch64 中，MAIR 寄存器存储了 8 种预定义的 8 位内存属性。页表项（PTE）中的 3 位 **AttrIndx** 字段，用于间接选择 MAIR 中的某一个属性（Attr0 到 Attr7）。

MAIR 寄存器是 64 位，由 8 个 8 位字段组成：
$$\text{MAIR}[63:0] = \{ \text{Attr}7, \text{Attr}6, \dots, \text{Attr}1, \text{Attr}0 \}$$

每个 $\text{Attr}x$ 字段结构：
$$\text{Attr}x[7:0] = \{ \underbrace{\text{Outer Attributes}}_{\text{4位}}, \underbrace{\text{Inner Attributes}}_{\text{4位}} \}$$

---

### 一、 内存类型划分 (Attr[7:6] 或 Attr[3:2])

内存类型由 4 位属性字段的最高 2 位决定：

| Attr[7:6] 或 Attr[3:2] | 内存类型 | 描述 |
| :--- | :--- | :--- |
| **0b00** | **Device Memory (设备内存)** | 访问 I/O 寄存器。**不可缓存**，顺序严格，禁止推测加载。 |
| **非 0b00** | **Normal Memory (普通内存)** | 访问主内存（RAM）。可以缓存，支持重排序和推测加载。 |

### 二、 缓存策略详解 (Normal Memory)

对于 **Normal Memory**，策略主要区分 **写策略** 和 **分配策略**。

#### 1. 写策略 (Write Policy)

| 策略 | 编码 (Attr[1:0] 或 Attr[5:4]) | 差异 |
| :--- | :--- | :--- |
| **Non-Cacheable** | **0b00** | 不可缓存。数据访问速度慢，但永恒一致。 |
| **Write-Through (WT)** | **0b10** | **写直通**。同时更新 Cache 和下一级内存。一致性高，但带宽占用大。 |
| **Write-Back (WB)** | **0b11** | **写回**。只更新 Cache Line，脏数据在替换或 Clean 时才写回。写入速度快。 |

#### 2. 分配策略 (Allocation Policy)

通常与 WT/WB 策略结合使用，定义在未命中时是否分配新的 Cache Line。

| 策略 | 描述 |
| :--- | :--- |
| **No-Allocate** | 读或写未命中时，**不**分配新的 Cache Line。 |
| **Read/Write-Allocate** | 读或写未命中时，**分配**新的 Cache Line。 |

#### 3. 瞬态性 (Transient)

定义 Cache 替换算法的偏好：

* **Transient (瞬态)：** 建议 Cache 替换时**优先淘汰**该 Line（如一次性数据）。
* **Non-transient (非瞬态)：** 建议 Cache **保留**该 Line（如关键程序数据）。

---

### 三、 MAIR 配置的常见示例

以下是常见的 MAIR 属性配置（AttrIndx 指向 MAIR 寄存器中的对应 Attr）：

| AttrIndx | 内存类型 | Inner Cache 策略 | Outer Cache 策略 | 典型用途 |
| :--- | :--- | :--- | :--- | :--- |
| **Attr0** | **Device** | Non-Cacheable | Non-Cacheable | I/O 寄存器访问。 |
| **Attr1** | **Normal** | **Non-Cacheable** | **Non-Cacheable** | 严格禁止缓存的共享数据。 |
| **Attr2** | **Normal** | WT, No-Allocate | WT, No-Allocate | 需要内存同步，但 Cache 利用率低的数据。 |
| **Attr3** | **Normal** | **WB, RW-Allocate** | **WB, RW-Allocate** | **默认高性能配置。** 用于大部分代码、堆、栈。 |
| **Attr4** | **Normal** | WB, Transient | WB, Transient | 临时缓冲区或一次性操作数据。 |
