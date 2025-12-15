### 1. 更新软件包索引

**目的**：确保系统能获取最新的软件包和驱动信息。  
**优势**：避免因软件源过期导致无法安装最新驱动或出现依赖错误。  
**隐患**：若网络不可靠，可能导致更新中断；但一般不会破坏系统。  
**指令**：

```bash
sudo apt update
```

---

### 2. 自动检测并安装推荐的 NVIDIA 驱动

**目的**：让系统自动选择与当前硬件最兼容、经过测试的官方闭源驱动版本。  
**优势**：省去手动查询型号和版本的步骤，降低误装风险；使用的是 Ubuntu 官方仓库验证过的稳定版本。  
**隐患**：

- 若网络中断，安装可能失败；
- 极少数情况下（如新显卡刚发布），推荐驱动可能不是最新版，但通常是最稳定的；
- 安装闭源驱动后若未重启，可能导致图形界面异常或黑屏。 **指令**：

```bash
sudo ubuntu-drivers autoinstall
sudo reboot
```

---

### 3. 验证 NVIDIA 驱动是否正常工作

**目的**：确认驱动已正确加载并能与 GPU 通信。  
**优势**：快速判断安装是否成功，避免后续使用中才发现问题（如 CUDA 无法调用 GPU）。  
**隐患**：若驱动未正确安装，命令会报错（如“NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver”），但不会造成系统损坏。  
**指令**：

```bash
nvidia-smi
```

---

### 4. （可选）手动查看可用驱动列表以确认推荐版本

**目的**：了解系统识别到的显卡型号及所有可用驱动选项。  
**优势**：便于用户理解系统为何选择某版本，或在需要特定版本（如用于深度学习）时做出调整。  
**隐患**：无实际操作风险，仅为信息查询。  
**指令**：

```bash
ubuntu-drivers devices
```

---

### 5. 检查 Secure Boot 状态（如遇驱动加载失败）

**目的**：Secure Boot 可能阻止未签名的第三方内核模块（如 NVIDIA 驱动）加载。  
**优势**：提前识别启动安全策略对驱动的影响，避免黑屏或 fallback 到开源驱动。  
**隐患**：

- 禁用 Secure Boot 会略微降低系统启动安全性；
- 若启用 Secure Boot 且未正确注册 MOK（Machine Owner Key），驱动将无法加载。 **检查指令**：

```bash
mokutil --sb-state
```

**建议**：

- 若返回 “SecureBoot enabled”，且驱动安装后无法使用，建议进入 BIOS 禁用 Secure Boot，或在安装驱动时按提示注册 MOK。

---

### 6. （可选）更换为国内软件源以加速下载

**目的**：提升 `apt` 下载速度，尤其在安装大型驱动包时。  
**优势**：显著缩短安装时间，提高成功率。  
**隐患**：

- 若镜像站同步延迟，可能暂时缺少最新包；
- 手动编辑 sources.list 存在语法错误风险（建议使用 GUI 工具或备份原文件）。 **操作方式**：
- 图形界面：打开“软件和更新” → “Ubuntu 软件” → “下载自” → 选择国内镜像（如阿里云、清华源）。
- 命令行（以清华源为例，需谨慎操作）：
    
    ```bash
    sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
    sudo sed -i 's|http://[a-z0-9\.]*\.archive\.ubuntu\.com|https://mirrors.tuna.tsinghua.edu.cn|g' /etc/apt/sources.list
    sudo apt update
    ```
    

---

### 总结建议顺序

1. 更新软件包索引（`apt update`）
2. 查看驱动推荐（`ubuntu-drivers devices`，可选）
3. 自动安装驱动（`ubuntu-drivers autoinstall`）
4. 重启系统
5. 验证驱动（`nvidia-smi`）
6. 如遇问题，检查 Secure Boot 状态或考虑更换软件源

此流程兼顾效率、稳定性与可排查性，适合绝大多数桌面和工作站用户。