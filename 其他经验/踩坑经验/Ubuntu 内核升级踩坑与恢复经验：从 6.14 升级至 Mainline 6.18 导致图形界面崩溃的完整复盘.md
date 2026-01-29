## 背景

为适配 NVIDIA RTX 5090 显卡（当时尚未正式支持），我尝试按照 AI 推荐的操作，将 Ubuntu 系统从默认内核 **6.14** 升级到 **Mainline 内核 6.18**。然而，该操作导致系统启动后图形界面完全崩溃，远程控制软件（如 AnyDesk、NoMachine）也无法运行，最终不得不前往现场进行手动修复。

本文记录此次事件的完整处理流程，包括问题复现、回滚旧内核、清理残留、优化引导配置等步骤，并附上关键命令，供后续参考。

---

## 一、问题现象

- 系统可正常进入 GRUB 或 rEFInd 引导菜单；
- 选择新内核 6.18 后，系统可完成内核加载；
- 但进入桌面环境失败（黑屏 / 登录循环 / GNOME Shell 崩溃）；
- 所有依赖图形界面的远程控制工具失效；
- TTY（Ctrl+Alt+F3）可登录，但无法通过网络远程操作图形界面。

---

## 二、紧急恢复：切换回旧内核

由于无法远程操作，需物理接触设备。进入引导菜单（rEFInd 或 GRUB）后，手动选择旧版内核（如 `6.14.x-xx-generic`）启动系统。

> 若使用 rEFInd，确保其配置中保留了旧内核的引导项（见第四部分优化建议）。

成功进入旧内核后，确认系统功能恢复正常。

---

## 三、彻底卸载 Mainline 内核 6.18

### 1. 查看已安装的内核版本

```bash
ls /boot/vmlinuz-*
# 或
dpkg -l | grep linux-image
```

确认存在类似 `linux-image-6.18.0-xxxx-generic` 的包。

### 2. 卸载指定内核及相关组件

```bash
sudo apt remove --purge \
    linux-image-6.18.0-xxxx-generic \
    linux-headers-6.18.0-xxxx-generic \
    linux-modules-6.18.0-xxxx-generic \
    linux-modules-extra-6.18.0-xxxx-generic
```

> 请将 `xxxx` 替换为实际版本号（可通过 `uname -r` 或 `dpkg -l` 查看）。

### 3. 清理无用依赖与更新 initramfs

```bash
sudo apt autoremove --purge
sudo update-initramfs -u -k all
```

### 4. 删除 Mainline 内核源（如果添加过）

若曾通过 [Ubuntu Mainline Kernel PPA](https://wiki.ubuntu.com/Kernel/MainlineBuilds) 安装，需移除相关源：

```bash
# 检查是否添加了 mainline 源
ls /etc/apt/sources.list.d/ | grep -i mainline

# 删除对应文件（示例）
sudo rm /etc/apt/sources.list.d/ubuntu-mainline-kernel-team-*.list

# 清理 GPG 密钥（如有）
sudo apt-key list | grep -A1 -B1 "mainline"
# 若找到密钥 ID（如 ABCDEF123456），执行：
sudo apt-key del ABCDEF123456

# 更新 apt 缓存
sudo apt update
```

> 注意：现代 Ubuntu 更推荐使用 `signed-by` 方式管理第三方源，而非 `apt-key`。此处仅为兼容旧方法。

---

## 四、优化 rEFInd 引导配置

为避免未来再次因默认启动新内核而“锁死”系统，对 `/boot/efi/EFI/refind/refind.conf` 进行如下优化：

### 1. 设置默认启动项为稳定内核

```conf
# 指定默认启动的条目（通过 volume + loader 路径匹配）
default_selection vmlinuz-6.14.0-xx-generic
```

> 可通过 `ls /boot/efi/EFI/ubuntu/` 查看实际文件名。

### 2. 增加超时并显示所有内核

```conf
timeout 20
scan_all_linux_kernels true
```

### 3. （可选）隐藏不稳定的测试内核

若某些内核明确不使用，可通过 `dont_scan_files` 排除：

```conf
dont_scan_files vmlinuz-6.18.0-xxxx-generic
```

保存后重启，验证 rEFInd 菜单是否按预期工作。

---

## 五、经验总结与反思

本次事故的根本原因在于：**盲目信任 AI 推荐，未充分评估 Mainline 内核的实验性质与硬件兼容性风险**。Mainline 内核虽包含最新驱动，但未经 Ubuntu 官方 QA 测试，尤其对闭源显卡驱动（如 NVIDIA）支持极不稳定。

### 关键教训：

- **不要在生产或主力机上直接使用 Mainline 内核**，除非你有备用方案或可接受系统不可用。
- **始终保留至少一个可工作的旧内核**，并在引导器中确保其可选。
- **AI 提供的建议应视为“参考”，而非“指令”** —— 尤其涉及系统底层变更时。

> 📌 **今后使用 AI 对话时，应主动要求其输出方案的优缺点、适用场景与潜在风险**，而非仅获取操作步骤。技术决策需结合自身环境权衡，而非一键执行。

---

希望这篇复盘能帮助他人避开类似陷阱。稳定压倒一切，尤其是在图形栈和内核这种核心组件上。