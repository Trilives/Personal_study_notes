## 一、背景说明

目标：在搭载 NVIDIA RTX 5090 显卡的新硬件平台上部署稳定版本的 Linux 系统（首选 Ubuntu 22.04 LTS）。

原因选择 Ubuntu 22.04：

- 长期支持（LTS），稳定性高；
- 社区文档丰富；
- 企业环境中广泛使用。

## 二、初始安装过程与问题

### 1. 安装介质准备

- 使用官方 Ubuntu 22.04.4 ISO 镜像（发布日期：2024年4月）；
- 通过 Rufus（Windows）或 dd 命令写入 USB 启动盘。

### 2. BIOS/UEFI 设置

- Secure Boot：已关闭；
- Fast Boot：已关闭；
- CSM（兼容性支持模块）：禁用，仅 UEFI 模式；
- Primary Display：设置为 PCIe（非板载显卡）。

### 3. 首次启动安装程序时黑屏

- 现象：进入 GRUB 菜单后选择 “Try or Install Ubuntu”，屏幕立即黑屏，无任何输出。
- 分析：RTX 5090 属于尚未被 Ubuntu 22.04 内核（默认 5.15）和开源 nouveau 驱动支持的新一代 GPU，nouveau 尝试初始化显卡失败导致显示异常。

### 4. 尝试内核参数绕过

在 GRUB 启动菜单按 `e` 编辑启动项，在 `linux` 行末尾添加以下参数组合进行测试：

```text
nomodeset
```

结果：可进入 Live 环境，但分辨率极低（800×600 或 1024×768），且无法启用图形加速。

进一步尝试：

```text
acpi=off
```

结果：系统可启动，但部分硬件（如 USB 控制器、电源管理）功能异常，不推荐长期使用。

最终采用组合参数成功进入 Live 环境：

```text
nomodeset acpi=off
```

## 三、安装 Ubuntu 22.04 系统

- 在 Live 环境中完成标准安装流程；
- 安装类型：清除整个磁盘并安装 Ubuntu；
- 未勾选“安装第三方软件”（因担心驱动冲突）；
- 安装完成后重启。

## 四、首次启动后的问题

### 1. 启动黑屏或低分辨率

- 重启后再次黑屏；
    
- 若能进入 TTY（Ctrl+Alt+F3），登录后检查：
    
    ```bash
    lspci | grep -i nvidia
    ```
    
    输出确认 GPU 被识别（如：NVIDIA Corporation Device 2684）。
    
- 图形界面无法正常加载，Xorg 日志报错：
    
    ```bash
    cat /var/log/Xorg.0.log | grep EE
    ```
    
    常见错误：`(EE) No devices detected` 或 `(EE) Failed to initialize GLX extension`。
    

### 2. 网络连接问题

- 主板集成网卡（如 Intel I225-V 或 Realtek RTL8125）在 Ubuntu 22.04 默认内核下驱动缺失或版本过旧；
- 无法通过有线网络联网；
- 临时解决方案：使用 Android 手机 USB 共享网络（USB tethering）。

启用 USB 网络共享步骤：

- 手机开启“USB 网络共享”；
- Ubuntu 中识别为新网络接口（通常为 enp0s20u2 或 usb0）；
- 自动获取 IP（DHCP）：
    
    ```bash
    ip a
    ```
    

### 3. 安装 NVIDIA 官方驱动

尝试多种方式安装驱动：

#### 方法 A：使用官方 .run 文件（不推荐）

- 下载 NVIDIA-Linux-x86_64-550.xx.run（适用于 Ada Lovelace 架构）；
- 禁用 nouveau：
    
    ```bash
    echo 'blacklist nouveau' | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
    echo 'options nouveau modeset=0' | sudo tee -a /etc/modprobe.d/blacklist-nouveau.conf
    sudo update-initramfs -u
    sudo reboot
    ```
    
- 进入 TTY，停止 gdm：
    
    ```bash
    sudo systemctl stop gdm3
    sudo chmod +x NVIDIA-Linux-x86_64-*.run
    sudo ./NVIDIA-Linux-x86_64-*.run
    ```
    
- 结果：驱动安装成功，但重启后仍黑屏，Xorg 无法启动。

#### 方法 B：使用 Ubuntu 官方仓库驱动

```bash
sudo ubuntu-drivers devices
# 输出建议驱动版本（如 nvidia-driver-535 或 550）
sudo apt update
sudo apt install nvidia-driver-550
sudo reboot
```

- 结果：重启后黑屏，驱动加载但 Xorg 配置失败。

#### 方法 C：使用 graphics-drivers PPA

```bash
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt update
sudo apt install nvidia-driver-550
```

- 结果：同上，无法解决显示问题。

### 4. USB 网络共享在重启后失效

- 安装驱动并重启后，USB 网络接口不再被识别；
- `ip a` 无 usb0 或 enp* 接口；
- 原因推测：内核模块（如 cdc_ether、rndis_host）因 nouveau 冲突或内核版本限制未正确加载；
- 尝试手动加载：
    
    ```bash
    sudo modprobe cdc_ether
    sudo modprobe rndis_host
    ```
    
    无效，dmesg 无相关设备事件。

## 五、根本原因分析

1. **内核版本过旧**  
    Ubuntu 22.04 默认内核为 5.15，而 RTX 5090（假设为 Ada Lovelace 后续架构）需要 Linux Kernel 6.6+ 及配套固件支持。
    
2. **NVIDIA 驱动与内核兼容性**  
    即使安装了最新版 NVIDIA 驱动（550+），其 DKMS 模块需匹配内核 ABI。Ubuntu 22.04 的内核头文件和编译环境不足以支持新 GPU 的完整初始化。
    
3. **主板新硬件支持不足**  
    新款主板（如 Z790/B650 等）的 USB、PCIe、NVMe 控制器在旧内核中缺乏完善驱动，导致 USB 网络、存储等子系统不稳定。
    

## 六、解决方案：转向 Ubuntu 24.04

### 1. 选择理由

- Ubuntu 24.04 默认搭载 Linux Kernel 6.8；
- 内置更新的 firmware-misc-nonfree 包，包含新 GPU 和网卡所需固件；
- 官方已预集成 NVIDIA driver 550+ 支持；
- 对 USB4/Thunderbolt、PCIe 5.0 等新特性原生支持更好。

### 2. 安装过程

- 使用 Ubuntu 24.04 Desktop ISO（2024年4月发布）；
- 启动时无需额外内核参数（或仅需 `nomodeset` 一次）；
- 安装过程中可直接联网（有线/USB tethering 正常）；
- 安装完成后自动识别 RTX 5090，通过“附加驱动”工具一键安装 NVIDIA 驱动；
- 重启后图形界面正常，分辨率正确，性能良好。

### 3. 验证命令（Ubuntu 24.04）

```bash
uname -r                          # 应为 6.8.x
nvidia-smi                        # 显示 GPU 信息
lspci -k | grep -A 3 -i nvidia    # 检查驱动绑定
ip a                              # 确认网络接口正常
```

## 七、结论

Ubuntu 22.04 虽为 LTS 版本，但其内核和驱动栈对 2024 年后发布的高端硬件（如 RTX 5090、新一代主板芯片组）支持不足。强行通过内核参数和手动驱动安装无法实现稳定运行。建议对于搭载最新 NVIDIA GPU 的设备，优先选用 Ubuntu 24.04 或更高版本，以获得开箱即用的硬件兼容性。

---

**记录人**：系统管理员  
**记录时间**：2025年12月10日  
**适用场景**：高性能计算工作站、AI 开发平台部署参考