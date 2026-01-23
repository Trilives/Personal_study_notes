以下是一份完整的 **Ubuntu 24.04 在搭载 RTX 5090 显卡的多 SSD 系统中安装与引导配置的技术解决笔记**。本笔记整合了从硬件识别、安装失败分析、BIOS 设置调整、驱动适配到双系统引导修复的全过程，适用于具有类似高端新硬件（如 RTX 50 系列）和复杂存储拓扑（多 NVMe/SATA SSD）的用户。

---

## 一、背景与问题概述

目标设备配置：

- 主板：MSI MS-7E47（支持 PCIe 5.0、UEFI）
- CPU：Intel 第 14 代（Raptor Lake Refresh）
- 显卡：NVIDIA GeForce RTX 5090（工程样品或早期零售版）
- 存储：两块 M.2 NVMe SSD（分别标识为 m2_1 和 m2_2），均用于系统或数据
- 原有系统：Windows 11 已安装在 m2_1 上
- 安装介质：Ubuntu 24.04 LTS Live USB（通过 Rufus 制作，GPT + UEFI 模式）

初始尝试使用 Ubuntu 22.04 完全失败，表现为：

1. 启动安装程序时黑屏或仅显示光标；
2. 即使通过 `acpi=off nomodeset` 参数进入试用模式，也无法完成图形界面安装；
3. 安装完成后无法识别 GRUB 引导项，始终直接启动 Windows；
4. 驱动更新后仍存在内核模块加载失败、分辨率异常等问题。

因此，转向 Ubuntu 24.04，并基于其更新的内核（6.8+）和更好的新硬件支持进行重试。

---

## 二、安装前准备：BIOS 关键设置

RTX 5090 属于尚未被主流 Linux 内核完全原生支持的新架构显卡，对主板固件兼容性要求极高。必须在 BIOS 中进行以下设置：

### 1. 启动模式

- **CSM（Compatibility Support Module）**：设为 **Disabled**  
    → 强制纯 UEFI 模式，避免 Legacy 兼容层干扰新 GPU 初始化。
- **Secure Boot**：设为 **Disabled**  
    → 避免未签名的 NVIDIA 驱动（尤其是 `.run` 安装包）被拒绝加载。
- **Fast Boot**：建议 **Disabled**（调试阶段）  
    → 确保所有硬件被完整枚举。

### 2. PCIe 与内存相关

- **Above 4G Decoding**：**Enabled**  
    → 允许系统为大显存（>4GB）GPU 分配连续高地址空间，对 RTX 5090 至关重要。
- **Resizable BAR (Re-Size BAR Support)**：**Enabled**（若主板支持）  
    → 提升 GPU 访问系统内存效率，Ubuntu 24.04 对此支持良好。
- **PCIe Speed**：设为 **Auto** 或 **Gen4**（避免强制 Gen5 导致不稳定）

### 3. Full Screen Logo

- **Full Screen Logo / Splash Screen**：**Disabled**  
    → 显示真实启动日志，便于排查卡顿位置（如 Nouveau 驱动崩溃、ACPI 错误等）。

---

## 三、安装过程关键操作

### 1. 识别目标磁盘（解决“E 盘”映射问题）

由于系统含多块 SSD（NVMe + SATA），且 U 盘会抢占 `/dev/sda`，不能依赖 Windows 的“磁盘 0/1/2”直接对应 `/dev/sda/sdb/sdc`。

**正确做法：**

- 在 Windows 中通过 `diskmgmt.msc` 确认目标盘为 **“磁盘 2”**。
- 进入 Ubuntu Live Session 后，打开 **GParted** 或终端执行：
    
    ```bash
    lsblk -d -o NAME,SIZE,ROTA,MODEL
    sudo fdisk -l
    ```
    
- 根据容量（如 1TB/2TB）和接口类型（NVMe 名为 `nvme0n1`，SATA 名为 `sdX`）匹配。
- **特别注意**：U 盘通常为第一个 SATA 设备（`/dev/sda`），因此 Windows “磁盘 2” 很可能对应 **`/dev/sdc`**（若为 SATA SSD）或 **`/dev/nvme1n1`**（若为第二块 NVMe）。
### 2. 确认目标磁盘（M.2_2）

由于系统含多个 NVMe SSD，且 U 盘会占用 `/dev/sda`，不能依赖 Windows 的“磁盘编号”直接映射。

**正确识别方法：**

- 在 Windows 中通过 `diskmgmt.msc` 确认 E 盘位于 **“磁盘 2”**。
- 进入 Ubuntu Live Session 后，打开终端执行：
 ```bash
    lsblk -d -o NAME,SIZE,MODEL
```
- 观察输出：
    - `/dev/nvme0n1`：通常对应 M.2_1（Windows 系统盘）
    - `/dev/nvme1n1`：对应 M.2_2（目标盘，容量与 E 盘一致）
    - `/dev/sda`：U 盘（容量小，可忽略）

> 注意：NVMe 设备命名独立于 SATA (`sdX`)，不受 U 盘插入顺序影响。

---

## 四、安装后首次启动问题：默认进 Windows

安装完成后重启直接进入 Windows，原因如下：

### 1. BIOS 启动优先级机制

多数主板（尤其 MSI、ASUS、Lenovo）有两个层级的启动控制：

- **Boot Device Priority**：选择从哪个设备启动（如 Hard Drive, USB, NVMe）
- **Hard Drive BBS Priorities**（或类似名称）：在“Hard Drive”类别下，**进一步指定哪块硬盘的 EFI 条目优先**

**错误操作**：仅将 “Boot Device Priority” 设为 “Hard Drive”，但未在 “Hard Drive BBS Priorities” 中将 `ubuntu` 条目置顶。

**正确操作**：

1. 进入 BIOS → Boot → Hard Drive BBS Priorities
2. 将 `ubuntu`（对应 m2_2）移动到第一位
3. 保存退出

### 2. GRUB 菜单被隐藏

即使 GRUB 已安装，若配置为自动启动默认项且超时为 0，则用户看不到菜单。

**修复方法**：

```bash
sudo nano /etc/default/grub
```

修改：

```ini
GRUB_TIMEOUT_STYLE=menu
GRUB_TIMEOUT=10
GRUB_DEFAULT=0
```

更新：

```bash
sudo update-grub
```

---

## 五、引导项重复问题：两个 Ubuntu 条目

执行 `sudo efibootmgr` 后发现：

- `Boot0002* Ubuntu` → 指向 Ubuntu 磁盘的 ESP（正确）
- `Boot0005* ubuntu` → 指向 Windows 磁盘的 ESP（多余残留）

**原因**：安装程序检测到多个可写 ESP，同时向两个分区写入了引导文件。

**清理步骤**：

1. 删除多余条目：
    
    ```bash
    sudo efibootmgr -b 0005 -B
    ```
    
2. 设置正确启动顺序：
    
    ```bash
    sudo efibootmgr -o 0002,0000,0004
    ```
    
    （顺序：Ubuntu → Windows → USB）

---

## 六、RTX 5090 驱动适配（后续关键步骤）

Ubuntu 24.04 自带驱动（如 `nvidia-driver-535`）可能不支持 RTX 5090。需手动升级：

### 1. 升级内核至 6.13+

- 使用 `mainline` 工具安装主线内核：
    
    ```bash
    sudo add-apt-repository ppa:cappelikan/ppa
    sudo apt update
    sudo apt install mainline
    ```
    
    启动 GUI 或命令行选择 6.13+ 内核安装。

### 2. 安装 GCC 14（匹配新内核编译环境）

```bash
sudo apt install gcc-14 g++-14
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-13 100
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-14 110
```

### 3. 安装最新 NVIDIA 驱动

- 从 [NVIDIA 官网](https://www.nvidia.com/Download/index.aspx) 下载 `.run` 文件（如 550+ 版本）
- 禁用 Nouveau：
    
    ```bash
    echo 'blacklist nouveau' | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
    sudo update-initramfs -u
    ```
    
- 重启进入文本模式（Ctrl+Alt+F3），停止显示管理器：
    
    ```bash
    sudo systemctl stop gdm3
    ```
    
- 安装驱动：
    
    ```bash
    sudo chmod +x NVIDIA-Linux-x86_64-*.run
    sudo ./NVIDIA-Linux-x86_64-*.run
    ```
    

---

## 七、经验总结

1. **新硬件需新系统**：RTX 5090 必须搭配 Ubuntu 24.04+（内核 ≥6.8），22.04 内核过旧，无法支持。
2. **BIOS 设置是成败关键**：CSM 关闭、Above 4G 开启、Secure Boot 关闭缺一不可。
3. **启动参数是救命稻草**：`nomodeset` 是绕过新 GPU 图形初始化失败的通用手段。
4. **磁盘识别靠容量，不靠名字**：在多 SSD 环境中，必须通过 `lsblk`/`fdisk`/GParted 结合容量判断目标盘。
5. **引导修复分两层**：既要调 BIOS 的 “Hard Drive BBS Priorities”，也要确保 GRUB 菜单可见。
6. **驱动需手动跟进**：官方仓库驱动滞后，必须升级内核 + GCC + 官方 `.run` 驱动三件套。

---

本笔记可作为在高端新平台部署 Ubuntu 的标准操作手册，尤其适用于 RTX 40/50 系列 + 多 NVMe SSD 的复杂场景。