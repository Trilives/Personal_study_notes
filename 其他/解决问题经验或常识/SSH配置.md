# Tailscale + SSH 部署与使用完整指南

## 1. 基础部署流程

### 1.1 前提条件
- 一台需要被 SSH 访问的 Linux 主机（如 Ubuntu/Debian/CentOS）
- 一个 Tailscale 账号（支持 Google/GitHub 登录）
- 本地设备（Windows/macOS/Linux/iOS/Android）

### 1.2 服务器端配置
1. 安装 Tailscale：
   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   ```

2. 启动并登录 Tailscale：
   ```bash
   sudo tailscale up
   ```
   执行后会输出授权 URL，用浏览器完成登录。

3. 查看分配的 Tailscale IP：
   ```bash
   tailscale ip
   # 或
   ip addr show tailscale0
   ```

4. 确保 SSH 服务正常运行：
   ```bash
   sudo apt install openssh-server    # Debian/Ubuntu
   sudo systemctl enable --now ssh
   ```

### 1.3 客户端配置
- 在本地设备安装 Tailscale 客户端
- 使用同一账号登录

### 1.4 连接测试
```bash
ssh username@100.x.y.z
```
其中 `username` 是服务器上的用户，`100.x.y.z` 是服务器的 Tailscale IP。

## 2. 传统 SSH 密钥 vs Tailscale SSH

### 2.1 传统 SSH
- 基于 `authorized_keys` 的公钥认证
- 依赖系统 `sshd` 服务
- 需要手动分发和管理密钥

### 2.2 Tailscale SSH
- 基于 Tailscale 账号身份认证
- 由 `tailscaled` 直接处理 SSH 连接，不依赖 `sshd`
- 无需配置 `authorized_keys`
- 支持 ACL 策略控制

### 2.3 迁移考虑
- 原有 SSH 密钥配置不需要删除，Tailscale SSH 不会使用它们
- 如果完全迁移到 Tailscale SSH，建议禁用传统 `sshd` 服务以提高安全性

## 3. 设备标签（Tag）管理

### 3.1 前提条件
- Tailscale Plus 或企业版账户（免费版不支持标签）

### 3.2 打标签步骤
1. 在 ACL 策略中授权标签所有权：
   ```json
   {
     "tagOwners": {
       "tag:webserver": ["your@email.com"]
     }
   }
   ```

2. 在设备上打标签：
   ```bash
   sudo tailscale up --advertise-tags=tag:webserver,tag:prod
   ```

3. 持久化配置（防止重启丢失）：
   ```bash
   sudo systemctl edit tailscaled
   ```
   添加：
   ```ini
   [Service]
   ExecStart=
   ExecStart=/usr/sbin/tailscaled --state=/var/lib/tailscale/tailscaled.state --socket=/run/tailscale/tailscaled.sock --advertise-tags=tag:my-server
   ```

### 3.3 在 SSH 规则中使用标签
```json
{
  "ssh": [
    {
      "action": "accept",
      "users": ["your@email.com"],
      "hosts": ["tag:ssh-server"],
      "as": "ubuntu"
    }
  ]
}
```

## 4. ACL 策略详解

### 4.1 基本字段说明
- **Source**: 允许发起连接的用户或设备组
- **Destination**: 允许连接的目标设备
- **As destination user**: 允许登录的目标用户身份
- **Check mode**: 是否启用安全检查
- **Check period**: 安全检查周期

### 4.2 内置特殊标识符
- `autogroup:self`: 用户自己拥有的设备
- `autogroup:members`: 所有已认证用户
- `autogroup:nonroot`: 所有非 root 的本地用户
- `autogroup:tagged`: 所有打了标签的设备

### 4.3 示例策略
只允许自己 SSH 自己的设备：
```json
{
  "ssh": [
    {
      "action": "accept",
      "users": ["*"],
      "hosts": ["autogroup:self"],
      "as": ["autogroup:nonroot", "root"]
    }
  ]
}
```

## 5. 传统 SSH 服务禁用

### 5.1 为什么需要禁用
- 减少攻击面
- 避免意外暴露 22 端口到公网
- 统一使用更安全的 Tailscale SSH

### 5.2 完整禁用步骤
现代 Linux 发行版使用 socket 激活机制，需要同时禁用 service 和 socket：

```bash
# 停止并禁用 socket
sudo systemctl stop ssh.socket
sudo systemctl disable ssh.socket

# 停止并禁用 service
sudo systemctl stop ssh.service
sudo systemctl disable ssh.service
```

### 5.3 验证是否彻底关闭
```bash
# 检查 22 端口
ss -tuln | grep ':22'

# 检查服务状态
systemctl is-active ssh.service
systemctl is-active ssh.socket
```

## 6. Web SSH 控制台说明

### 6.1 什么是 Web SSH
- Tailscale 提供的浏览器内 SSH 终端
- 用于快速访问和紧急调试
- 会话名称格式：`tailscale-ssh-console-username-provider-randomstring`

### 6.2 临时设备限制
- Web SSH 会创建临时设备（ephemeral device）
- 免费账户限制：3 台常规设备 + 1 台临时设备 = 4 台在线设备
- 超过限制后新设备无法加入网络

### 6.3 最佳实践
- 优先使用本地终端 SSH 连接
- Web SSH 仅用于紧急情况
- 使用后及时断开连接以释放临时设备名额

## 7. SSH 用户身份说明

### 7.1 用户身份要求
- SSH 登录必须使用目标机器上已存在的本地用户
- SSH 协议本身不会创建新用户
- 用户必须存在于 `/etc/passwd` 中

### 7.2 Tailscale SSH 用户映射
ACL 配置中的 `"as"` 字段指定允许使用的本地用户：
```json
{
  "ssh": [{
    "users": ["you@gmail.com"],
    "hosts": ["*"],
    "as": ["ubuntu", "root"]
  }]
}
```
- `ubuntu` 和 `root` 必须是目标服务器上真实存在的用户
- 如果用户不存在，即使 ACL 允许也会登录失败

### 7.3 用户管理建议
- 创建专用用户（如 `deploy`、`ops`）用于远程访问
- 避免直接使用 `root` 登录
- 在目标服务器上预先创建所需用户：
  ```bash
  sudo adduser deploy
  ```

## 8. 故障排除

### 8.1 常见问题
- **"You are not allowed to advertise tag"**: 未在 ACL 的 `tagOwners` 中授权标签
- **SSH 连接被拒绝**: 检查 ACL 策略、用户是否存在、Tailscale SSH 是否启用
- **设备数超限**: 关闭不用的 Web SSH 会话，或升级到 Plus 版本

### 8.2 验证命令
- 检查 Tailscale SSH 状态：
  ```bash
  sudo tailscale status --peers=false
  ```
- 查看用户是否存在：
  ```bash
  id username
  getent passwd
  ```

## 9. 安全最佳实践

1. **完全迁移到 Tailscale SSH** 并禁用传统 `sshd`
2. **使用 `autogroup:self`** 限制用户只能访问自己的设备
3. **启用 Check mode** 进行设备安全状态检查
4. **避免使用 root**，创建专用用户进行操作
5. **定期审查 ACL 策略**，遵循最小权限原则
6. **优先使用本地终端** 而非 Web SSH 控制台

## 10. 参考资源

- Tailscale 官方文档: https://tailscale.com/kb/
- ACL 策略编辑器: https://login.tailscale.com/admin/acls
- 设备管理页面: https://login.tailscale.com/admin/machines
- 定价信息: https://tailscale.com/pricing