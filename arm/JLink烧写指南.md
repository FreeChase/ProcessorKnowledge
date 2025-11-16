
J-Link Commander 中的命令非常多，可以分为几大类。以下是一些最常用和最重要的命令：

---

### 一、基本操作与连接命令

| 命令（长/短） | 描述 |
| :--- | :--- |
| **`?`** | 显示所有命令的列表和简短描述。 |
| **`Exit`** | 关闭 J-Link 连接并退出 J-Link Commander。 |
| **`Connect`** (`Con`) | 尝试连接到目标设备。 |
| **`Device <name>`** | 选择要连接的目标 CPU 或设备型号（如 `STM32F405RG`）。 |
| **`SelectInterface <interface>`** (`SI`) | 选择目标接口（如 `JTAG`、`SWD`）。 |
| **`Speed <value>`** | 设置目标接口的速度（单位：kHz）。 |
| **`Halt`** (`H`) | 暂停（停止）目标 CPU 的执行。 |
| **`Go`** (`G`) | 恢复目标 CPU 的执行。 |
| **`Reset`** (`R`) | 重置目标 CPU。 |
| **`Power [on/off]`** | 控制 J-Link 5V/VTref 引脚的电源输出（如果支持）。 |

---

### 二、烧录与文件操作命令

| 命令（长/短） | 描述 |
| :--- | :--- |
| **`loadfile <filename>[, <address>]`** | 将文件（通常是 `.hex`, `.bin`, `.elf` 等）烧录到目标 Flash 或 RAM 中。**（这就是您遇到问题的命令）** |
| **`loadbin <filename>, <address>`** | 专门用于将原始二进制文件 (`.bin`) 烧录到指定地址。 |
| **`erase`** | 擦除目标 Flash 存储器（通常是全片擦除）。 |
| **`savebin <filename>, <address>, <size>`** | 将目标存储器中的指定地址和大小的内容读取并保存为原始二进制文件。 |

---

### 三、内存和寄存器操作命令

| 命令（长/短） | 描述 |
| :--- | :--- |
| **`Mem <address>, <num_bytes>`** (`M`) | 以字节格式转储（读取）指定地址开始的一块内存。 |
| **`Mem32 <address>, <num_words>`** | 以 32 位字格式转储（读取）指定地址开始的一块内存。 |
| **`Wr <address>, <value>`** | 写入单个字节到指定地址。 |
| **`Wr32 <address>, <value>`** | 写入单个 32 位字到指定地址。 |
| **`Regs`** | 显示所有 CPU 寄存器的值。 |
| **`SetReg <reg_name>, <value>`** | 设置指定寄存器的值（如 `SetReg PC, 0x08000000`）。 |

---

### 四、高级与配置命令

| 命令（长/短） | 描述 |
| :--- | :--- |
| **`Log <filename>`** | 启用日志记录，将 J-Link 的所有输出保存到指定文件。 |
| **`rstat`** | 显示 J-Link 与目标芯片的运行时状态（如 CPU 状态、连接速度）。 |
| **`Lock`** | 启用目标 Flash 锁定（Read-Out Protection，取决于设备）。 |
| **`Unlock`** | 尝试解除 Flash 锁定/读保护（取决于设备和保护级别）。 |

通常，烧录固件的完整流程会在 J-Link Commander 中使用以下一系列命令：

```bash
JLinkExe 
# 进入 Commander 命令行界面

J-Link> si swd              # 选择 SWD 接口
J-Link> speed 4000          # 设置速度为 4000 kHz
J-Link> device STM32F405RG  # 选择目标设备型号 alternative choose cortex-A7 cortex-M3
J-Link> connect             # 连接到目标

J-Link> erase               # (可选) 擦除 Flash
J-Link> loadfile mcuf405.hex, 0x08000000 # 烧录文件
J-Link> reset               # 复位芯片
J-Link> go                  # 运行程序

J-Link> exit
````

