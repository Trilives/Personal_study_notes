# SSH 密钥配置与安全登录实践笔记（含 Tailscale 公网穿透）

## 背景说明

目标：从 Windows 本地机器通过 SSH 安全登录到远程 Ubuntu 服务器（`trlives@100.72.134.25`，主机名 `MS-7E47-5090`），同时满足以下条件：

- 保留默认 SSH 密钥用于 GitHub；
- 服务器已禁用密码认证；
- **服务器位于内网或无公网 IP，需通过 Tailscale 实现安全穿透访问**。

---

## 一、禁止 root 登录 SSH 的利弊分析

### 好处
1. **降低暴力破解风险**：攻击者需同时猜测用户名和密码，而非仅密码。
2. **遵循最小权限原则**：日常操作使用普通用户，仅在必要时通过 `sudo` 提权。
3. **增强审计能力**：操作可追溯至具体用户，而非统一的 `root`。
4. **支持精细权限控制**：通过 `/etc/sudoers` 限制命令范围。
5. **便于集成多因素认证（MFA）**：可在 `sudo` 提权时触发二次验证。

### 坏处与注意事项
1. **配置不当可能导致失联**：需确保至少一个普通用户具备 SSH 密钥登录和 `sudo` 权限。
2. **紧急恢复依赖控制台**：系统故障时需通过云平台 VNC 或串口访问。
3. **自动化脚本需调整**：原以 root 运行的任务需改用普通用户 + sudo。

### 是否可以使用 `sudo`
- 完全可以，且是推荐做法。
- 流程：普通用户 SSH 登录 → 执行 `sudo command` 或 `sudo -i` 提权。
- 用户需加入 `sudo`（Ubuntu/Debian）或 `wheel`（RHEL/CentOS）组。

---

## 二、Tailscale 实现内网穿透

### 2.1 为什么使用 Tailscale？
- 无需公网 IP、端口映射或动态 DNS；
- 基于 WireGuard 协议，加密安全；
- 自动 NAT 穿透，跨平台支持（Windows/Linux/macOS）；
- 免费个人使用（最多 3 台设备进 Access Control，更多需付费或自建 headscale）。

### 2.2 部署步骤

#### 在服务器（Ubuntu）上安装 Tailscale
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```
- 执行后会输出一个登录链接，用浏览器打开并授权；
- 设备将出现在 [https://login.tailscale.com/admin/machines](https://login.tailscale.com/admin/machines)。

#### 在 Windows 本地安装 Tailscale
1. 下载安装：https://tailscale.com/download/windows
2. 启动应用并登录同一账户；
3. 两台设备自动建立加密隧道。

#### 获取服务器的 Tailscale IP
在服务器执行：
```bash
tailscale ip -4
# 示例输出：100.x.y.z
```
该 IP 为私有虚拟 IP，仅 Tailscale 网络内可达。

> 注意：此 IP 与你之前使用的 `100.72.134.25`（可能是 CGNAT 或局域网 IP）不同。若原 IP 不可达，应改用 Tailscale IP。

---

## 三、SSH 配置适配 Tailscale

### 3.1 修改 SSH 客户端配置

编辑 `C:\Users\<用户名>\.ssh\config`，使用 Tailscale IP：

```
Host ubuntu-5090
    HostName 100.x.y.z          # 替换为服务器的 Tailscale IPv4 地址
    User trlives
    IdentityFile ~/.ssh/id_ed25519_ubuntu_5090
    IdentitiesOnly yes
    Port 22
```

### 3.2 优势
- **无需暴露 SSH 到公网**：即使服务器在 NAT 后、防火墙后，也能安全访问；
- **端到端加密**：Tailscale 已加密，SSH 再次加密，双重保障；
- **固定虚拟 IP**：Tailscale IP 长期稳定（除非重装或重认证）。

---

## 四、Windows 下无 `ssh-copy-id` 的替代方案

`ssh-copy-id` 是 Linux/macOS 工具，Windows 默认不提供。等效操作为手动将公钥追加至服务器的 `~/.ssh/authorized_keys`。

### 推荐流程（PowerShell）
1. 生成密钥对：
   ```powershell
   ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\id_ed25519_ubuntu_5090" -C "windows-laptop-for-trlives-server"
   ```
2. 获取公钥内容：
   ```powershell
   Get-Content "$env:USERPROFILE\.ssh\id_ed25519_ubuntu_5090.pub"
   ```
3. 通过服务器控制台（因密码登录已禁用）将公钥追加到 `~/.ssh/authorized_keys`。

---

## 五、服务器端操作要点

### 公钥文件位置
- SSH 服务默认读取 `~/.ssh/authorized_keys`。
- 使用 `>>` 追加公钥不会覆盖已有内容，适合多设备管理。
- 错误示例：写入 `authorized_keys_1` 无效，除非修改 `sshd_config` 中的 `AuthorizedKeysFile`。

### 权限设置（关键）
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```
权限过宽会导致 OpenSSH 拒绝使用该文件。

---

## 六、Windows 客户端完整配置流程（PowerShell）

### 步骤 1：生成专用密钥（避免与 GitHub 冲突）
```powershell
New-Item -Path "$env:USERPROFILE\.ssh" -ItemType Directory -Force
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\id_ed25519_ubuntu_5090" -C "windows-for-5090" -N ""
```

### 步骤 2：获取公钥并手动添加到服务器
- 复制公钥内容：
  ```powershell
  Get-Content "$env:USERPROFILE\.ssh\id_ed25519_ubuntu_5090.pub"
  ```
- 通过服务器控制台执行：
  ```bash
  echo "ssh-ed25519 AAAA... windows-for-5090" >> ~/.ssh/authorized_keys
  chmod 600 ~/.ssh/authorized_keys
  ```

### 步骤 3：配置 SSH 别名（使用 Tailscale IP）
创建 `C:\Users\<用户名>\.ssh\config`：
```
Host ubuntu-5090
    HostName 100.x.y.z          # Tailscale 分配的 IPv4
    User trlives
    IdentityFile ~/.ssh/id_ed25519_ubuntu_5090
    IdentitiesOnly yes
```

### 步骤 4：测试登录
```powershell
ssh ubuntu-5090
```

---

## 七、安全加固建议

### 客户端（Windows）
- 设置私钥文件 ACL，仅允许当前用户读取：
  ```powershell
  $user = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name
  icacls "$env:USERPROFILE\.ssh\id_ed25519_ubuntu_5090" /inheritance:r
  icacls "$env:USERPROFILE\.ssh\id_ed25519_ubuntu_5090" /grant:r "$user:(R)"
  ```
- 使用 SSH Agent 缓存 passphrase：
  ```powershell
  Start-Service ssh-agent
  ssh-add "$env:USERPROFILE\.ssh\id_ed25519_ubuntu_5090"
  ```

### 服务端（Ubuntu）
- 确保 `/etc/ssh/sshd_config` 包含：
  ```
  PasswordAuthentication no
  PermitRootLogin no
  PubkeyAuthentication yes
  AuthorizedKeysFile .ssh/authorized_keys
  ```
- 重载配置：
  ```bash
  sudo systemctl reload sshd
  ```
- **可进一步限制 SSH 仅监听 Tailscale 接口**（可选）：
  ```conf
  ListenAddress 100.x.y.z
  ```
  或使用防火墙仅允许 Tailscale 子网访问 SSH 端口。

---

## 八、其他说明

### 关于 ESM（Expanded Security Maintenance）提示
- Ubuntu 24.04 LTS 默认提供 5 年基础安全更新。
- ESM 可额外提供 5 年 `universe` 仓库的安全更新（总计 10 年）。
- 个人用户可免费注册 Ubuntu Pro（最多 5 台机器）：
  ```bash
  sudo pro attach
  ```

### 密钥管理最佳实践
- 一个用途一个密钥（如 GitHub、服务器分离）。
- 为私钥设置强 passphrase。
- 避免私钥多处分发；若需多设备访问，应为每台设备生成独立密钥对。

### Tailscale 高级用法（可选）
- **MagicDNS**：可通过 `hostname.tailnet-name.ts.net` 访问设备，无需记 IP。
- **Subnet Router**：可将整个子网接入 Tailscale 网络。
- **Exit Node**：将流量出口设为服务器，实现远程代理。

---

## 总结

通过 **Tailscale + SSH 密钥认证** 的组合，可在不暴露 SSH 服务到公网的前提下，实现安全、稳定、跨网络的远程管理。整个方案：

- 无需公网 IP 或端口转发；
- 端到端加密，符合零信任安全模型；
- 兼容现有 SSH 工作流（如 `scp`、`rsync`、VS Code Remote-SSH）；
- 适用于家庭实验室、边缘设备、云服务器等多种场景。

此架构兼顾安全性、易用性与可维护性，是现代远程运维的理想选择。