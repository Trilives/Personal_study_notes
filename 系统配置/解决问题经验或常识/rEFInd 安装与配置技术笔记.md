## 1. 安装 rEFInd

### 1.1 确认系统为 UEFI 模式

执行以下命令确认：

```bash
ls /sys/firmware/efi
```

若输出目录内容，则系统运行在 UEFI 模式，可继续安装。

### 1.2 通过包管理器安装（推荐）

#### Ubuntu/Debian 系列：

```bash
sudo apt update
sudo apt install refind
```

安装过程中提示是否将 rEFInd 安装到 EFI 分区时，选择 Yes。

#### Fedora/RHEL/CentOS Stream：

```bash
sudo dnf install refind
sudo refind-install
```

#### Arch Linux/Manjaro：

```bash
sudo pacman -S refind
sudo refind-install
```

### 1.3 手动安装（适用于无软件包或需最新版）

1. 下载官方压缩包（以 0.14.0.2 版本为例）：
    
    ```bash
    wget https://sourceforge.net/projects/refind/files/0.14.0.2/refind-bin-0.14.0.2.zip
    unzip refind-bin-0.14.0.2.zip
    cd refind-bin-0.14.0.2
    ```
    
2. 执行安装脚本：
    
    ```bash
    sudo ./refind-install
    ```
    
    脚本会自动将文件复制到 `/boot/efi/EFI/refind/` 并注册为 EFI 启动项。
    
3. 重启验证：
    
    ```bash
    sudo reboot
    ```
    

---

## 2. 配置文件位置与编辑方式

配置文件路径：

```
/boot/efi/EFI/refind/refind.conf
```

**注意：该文件是 rEFInd 的专用配置文件，不能在终端中直接执行。**

正确编辑方式：

```bash
sudo nano /boot/efi/EFI/refind/refind.conf
# 或使用图形编辑器（如在桌面环境中）
sudo gedit /boot/efi/EFI/refind/refind.conf
```

---

## 3. 精简启动菜单

### 3.1 屏蔽重复或不需要的引导项

在 `refind.conf` 中添加以下指令：

- **屏蔽特定 EFI 目录**（防止扫描出多个重复入口）：
    
    ```conf
    dont_scan_dirs EFI/ubuntu,EFI/debian,EFI/fedora,EFI/boot
    ```
    
- **屏蔽内核文件单独显示**（避免每个 vmlinuz 都列为一项）：
    
    ```conf
    dont_scan_files vmlinuz,initrd.img
    ```
    
- **折叠旧版 Linux 内核**（仅显示最新内核，其余可按 Insert 键展开）：
    
    ```conf
    fold_linux_kernels true
    ```
    

### 3.2 控制工具项显示

减少底部工具栏项目，例如只保留重启和关机：

```conf
showtools reboot,shutdown
```

---

## 4. 设置自动启动（倒计时）

### 4.1 启用倒计时并指定秒数

取消注释或添加以下行：

```conf
timeout 5
```

- 数值单位为秒。
- `0` 表示立即启动默认项（不显示菜单）。
- `-1` 表示无限等待（默认行为）。

### 4.2 指定默认启动项（可选）

rEFInd 默认启动检测到的第一个系统。可通过关键词匹配指定默认项：

```conf
default_selection ubuntu
```

该配置会匹配启动项名称中包含 “ubuntu”（不区分大小写）的条目。

> 提示：先正常启动一次 rEFInd，观察目标启动项显示的文本，提取唯一关键词用于 `default_selection`。

---

## 5. 可选：使用内置界面风格

### 5.1 启用金属图标主题

在 `refind.conf` 中添加：

```conf
include metal.conf
```

### 5.2 启用自带背景图

确保 `/boot/efi/EFI/refind/background.png` 存在，然后启用：

```conf
background background.png
```

### 5.3 切换为纯文本模式（极简）

```conf
textonly
```

此模式关闭所有图形元素，仅显示文本列表，适合低分辨率或快速启动场景。

---

## 6. 常见问题排查

### 6.1 安装后未显示 rEFInd 菜单

- 进入 BIOS/UEFI 设置，确认启动模式为 **UEFI Only**（禁用 Legacy/CSM）。
- 在启动选项中手动将 **rEFInd** 设为第一启动项。
- 某些品牌主板（如 Dell、Lenovo）可能自动清理非 Windows 启动项，可使用以下命令重新注册：
    
    ```bash
    sudo efibootmgr -v          # 查看当前启动项
    sudo refind-install         # 重新安装并注册
    ```
    

### 6.2 修改配置无效

- 确保编辑的是 `/boot/efi/EFI/refind/refind.conf`，而非其他副本。
- 修改后必须重启才能生效。
- 若配置语法错误导致无法启动，可使用 Live USB 挂载 ESP 分区后修复。

---

## 7. 验证安装结果

检查关键文件是否存在：

```bash
ls /boot/efi/EFI/refind/
```

应包含：

- `refind_x64.efi`
- `refind.conf`
- `icons/` 目录
- `background.png`（可选）

重启后应看到 rEFInd 图形菜单，并按配置自动倒计时启动。