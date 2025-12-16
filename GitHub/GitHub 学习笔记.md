# GitHub 学习笔记（Ubuntu / Windows）：SSH/HTTPS 配置、同步仓库与常用指令

> 目标：掌握两种与 GitHub 交互方式（HTTPS / SSH），能在 Ubuntu 和 Windows 上完成：
> - 安装与基础配置
> - 生成并配置 SSH Key
> - 查看本机已有 SSH Key / ssh-agent 已加载的 Key
> - 用 HTTPS/SSH 克隆、拉取、推送、同步
> - 常用 Git 指令速查
> - 常见问题排查
> - （附）子模块 submodule 常用操作

---

## 0. 术语快速理解

- **remote**：远程仓库地址（常见默认名 `origin`）
- **clone**：把远程仓库下载到本地
- **fetch**：只下载远程更新，不改工作区
- **pull**：`fetch + merge/rebase`，下载并合并到当前分支
- **push**：把本地提交推送到远程
- **upstream**：你 fork 来的上游仓库（常用于保持 fork 同步）
- **SSH** vs **HTTPS**：
  - HTTPS：简单，适合新手；推送一般需要 **PAT(Token)**（不再支持账户密码）
  - SSH：配置一次后免输入 token；更适合长期使用

---

## 1. 安装 Git

### Ubuntu
```bash
sudo apt update
sudo apt install -y git
git --version
```

### Windows
推荐安装 **Git for Windows**（自带 Git Bash）：
- 安装后打开 **Git Bash** 或 PowerShell：
```bash
git --version
```

---

## 2. Git 基础全局配置（用户名与邮箱）

> 提交记录会显示这些信息（建议与你的 GitHub 账号一致）

### Ubuntu / Windows（通用）
```bash
git config --global user.name "YourName"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
```

查看配置：
```bash
git config --global -l
```

可选：换行处理（建议保持默认即可）
- Windows 建议：
```bash
git config --global core.autocrlf true
```
- Ubuntu/macOS 建议：
```bash
git config --global core.autocrlf input
```

---

## 3. 使用 HTTPS 与 GitHub 同步仓库（无需 SSH）

### 3.1 克隆（HTTPS）
```bash
git clone https://github.com/<owner>/<repo>.git
cd <repo>
```

### 3.2 推送时的身份验证（GitHub Token / PAT）
GitHub 已不支持 HTTPS 使用账号密码推送，需要 Personal Access Token（PAT）。

创建 PAT（在 GitHub 网页）：
- Settings -> Developer settings -> Personal access tokens
- 选择合适权限（常见：repo）

在命令行推送：
```bash
git push
```

第一次会提示输入用户名与密码：
- 用户名：GitHub 用户名
- 密码：粘贴 **PAT token**（不是你的 GitHub 登录密码）

#### Windows 常见：凭据缓存
Windows 通常会用系统凭据管理器保存 token。也可以手动启用：
```bash
git config --global credential.helper manager
```

#### Ubuntu 常见：凭据缓存（可选）
```bash
git config --global credential.helper store
# 或缓存一段时间（秒）
git config --global credential.helper 'cache --timeout=3600'
```

---

## 4. 使用 SSH 与 GitHub 同步仓库（推荐长期使用）

### 4.1 前置检查：是否已安装 OpenSSH

#### Ubuntu
一般自带；没有则安装：
```bash
sudo apt update
sudo apt install -y openssh-client
ssh -V
```

#### Windows
- Windows 10/11 通常内置 OpenSSH
- 检查：
```powershell
ssh -V
```
如果没有，可在“可选功能”里安装 OpenSSH Client，或使用 Git for Windows 自带 ssh（Git Bash）。

---

## 5. 查看当前系统已有的 SSH Key（重点补充）

> SSH Key 默认存放在 **~/.ssh/** 目录。常见文件：
> - 私钥：`id_ed25519` / `id_rsa`
> - 公钥：`id_ed25519.pub` / `id_rsa.pub`
>
> 注意：**私钥不要泄露**（如 `id_ed25519`、`id_rsa`），可以安全分享的是 `*.pub` 公钥。

### 5.1 Ubuntu：查看 `~/.ssh` 目录下已有 key 文件

```bash
# 查看目录是否存在
ls -ld ~/.ssh

# 列出所有 ssh 相关文件（更详细）
ls -al ~/.ssh

# 只列出常见 key 文件（存在才会显示）
ls -al ~/.ssh/id_* ~/.ssh/*.pub 2>/dev/null
```

查看已有公钥内容（用于粘贴到 GitHub）：
```bash
cat ~/.ssh/id_ed25519.pub
# 或
cat ~/.ssh/id_rsa.pub
```

### 5.2 Windows：查看已有 key 文件

#### 方式 A：Git Bash
```bash
ls -al ~/.ssh
ls -al ~/.ssh/id_* ~/.ssh/*.pub 2>/dev/null
cat ~/.ssh/id_ed25519.pub
```

#### 方式 B：PowerShell
```powershell
# 列出 .ssh 目录
Get-ChildItem -Force $env:USERPROFILE\.ssh

# 只看公钥文件
Get-ChildItem -Force $env:USERPROFILE\.ssh\*.pub

# 输出公钥内容
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
# 或
Get-Content $env:USERPROFILE\.ssh\id_rsa.pub
```

### 5.3 查看 ssh-agent 当前“已加载”的 key（不是磁盘上所有 key）

> 有些 key 存在于 `~/.ssh`，但没有被 `ssh-agent` 加载；此时连接可能仍然失败。

Ubuntu / Git Bash（通用）：
```bash
ssh-add -l
```

如果输出类似 “The agent has no identities.” 表示当前没加载任何 key。

加载 key：
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
# 或
ssh-add ~/.ssh/id_rsa
```

Windows PowerShell：
```powershell
ssh-add -l
# 加载 key
ssh-add $env:USERPROFILE\.ssh\id_ed25519
```

---

## 6. 创建 SSH Key 并添加到 GitHub

> 建议使用 **ed25519**（更现代）；如果环境不支持再用 rsa

---

### 6.1 Ubuntu 创建 SSH Key（ed25519）

```bash
# 1) 生成 key（把邮箱替换成你的 GitHub 邮箱）
ssh-keygen -t ed25519 -C "you@example.com"

# 提示保存路径：默认回车 -> ~/.ssh/id_ed25519
# 提示 passphrase：可设置（更安全），也可直接回车为空
```

启动 ssh-agent 并添加私钥：
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

复制公钥：
```bash
cat ~/.ssh/id_ed25519.pub
```

---

### 6.2 Windows 创建 SSH Key（Git Bash 方式，推荐）

打开 **Git Bash**：

```bash
ssh-keygen -t ed25519 -C "you@example.com"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub
```

### 6.3 Windows 创建 SSH Key（PowerShell 方式）

```powershell
ssh-keygen -t ed25519 -C "you@example.com"
# 默认路径通常在 C:\Users\<you>\.ssh\id_ed25519

# 启动 ssh-agent 服务（可能需要管理员权限）
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent

ssh-add $env:USERPROFILE\.ssh\id_ed25519
type $env:USERPROFILE\.ssh\id_ed25519.pub
```

---

### 6.4 在 GitHub 添加 SSH Key（网页）

GitHub:
- Settings -> SSH and GPG keys -> New SSH key
- 粘贴 `id_ed25519.pub` 的内容
- Title 可写 “My laptop” / “Company PC” 等

---

### 6.5 验证 SSH 连接

Ubuntu / Windows（通用）：
```bash
ssh -T git@github.com
```

成功典型输出：
- `Hi <username>! You've successfully authenticated...`

---

## 7. 使用 SSH 地址进行 clone / push / pull

### 7.1 Clone（SSH）
```bash
git clone git@github.com:<owner>/<repo>.git
cd <repo>
```

### 7.2 查看远程地址
```bash
git remote -v
```

### 7.3 HTTPS 与 SSH 互相切换

#### 切换为 SSH
```bash
git remote set-url origin git@github.com:<owner>/<repo>.git
git remote -v
```

#### 切换为 HTTPS
```bash
git remote set-url origin https://github.com/<owner>/<repo>.git
git remote -v
```

---

## 8. 最常用 Git 指令（日常开发必备）

### 8.1 仓库与状态
```bash
git status
git log --oneline --decorate --graph --all
git diff
git diff --staged
```

### 8.2 添加与提交
```bash
git add <file>
git add .
git commit -m "your message"
```

修改上一次提交信息（未 push 前常用）：
```bash
git commit --amend
```

### 8.3 分支操作
```bash
git branch
git branch -a
git switch -c feature/login   # 新建并切换
git switch main               # 切换分支（旧命令：git checkout）
```

合并分支：
```bash
git switch main
git merge feature/login
```

删除本地分支：
```bash
git branch -d feature/login
# 强制删除：
git branch -D feature/login
```

### 8.4 拉取与推送（同步远端）

拉取当前分支：
```bash
git pull
```

推送当前分支：
```bash
git push
```

首次推送新分支（建立 upstream 追踪）：
```bash
git push -u origin feature/login
```

只下载更新（不合并）：
```bash
git fetch --all --prune
```

---

## 9. 同步 fork：保持你的仓库跟上 upstream

> 场景：你 fork 了别人的仓库到自己账户（origin），还想同步原仓库（upstream）的更新。

### 9.1 添加 upstream
```bash
git remote -v
git remote add upstream https://github.com/<upstream-owner>/<repo>.git
# 或 SSH:
# git remote add upstream git@github.com:<upstream-owner>/<repo>.git

git remote -v
```

### 9.2 同步 upstream 的 main 到本地 main
```bash
git fetch upstream
git switch main
git merge upstream/main
# 或用 rebase（更线性）：
# git rebase upstream/main
```

推送同步到你的 fork（origin）：
```bash
git push origin main
```

---

## 10. 常见问题排查

### 10.1 SSH 连接报错：Permission denied (publickey)
检查：
1. GitHub 是否添加了正确的 `*.pub`
2. 本机是否加载 key：
   ```bash
   ssh-add -l
   ```
3. 直接测试：
   ```bash
   ssh -vT git@github.com
   ```
4. 如果你有多个 key，可配置 `~/.ssh/config`（示例见下）

---

### 10.2 多个 SSH Key（公司/个人）配置示例

```
# 文件：~/.ssh/config

Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
```

如果要为不同账号使用不同 host：
```
Host github-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_personal

Host github-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_work
```

则远程地址使用：
- `git@github-personal:<owner>/<repo>.git`
- `git@github-work:<owner>/<repo>.git`

---

### 10.3 HTTPS 推送失败：要求输入密码但不接受
原因：GitHub 不支持用登录密码进行 HTTPS Git 操作。
解决：使用 PAT token 或改用 SSH。

---

## 11. （附）子模块 submodule 常用指令

### 11.1 添加子模块
```bash
git submodule add <子仓库URL> <路径>
git add .gitmodules <路径>
git commit -m "Add submodule"
```

### 11.2 克隆带子模块的仓库
```bash
git clone --recurse-submodules <主仓库URL>
```

或已经 clone 后：
```bash
git submodule update --init --recursive
```

### 11.3 更新子模块
```bash
git submodule update --remote --recursive
```

---

## 12. 推荐学习路径（练习清单）

1. 安装 Git + 设置 user.name/user.email
2. 先用 HTTPS clone 一个仓库、提交一次、push
3. 再配置 SSH key，切换 origin 到 SSH 并 push
4. 学会：branch / commit / pull / push / merge
5. 学会：fork 仓库 + upstream 同步
6. 最后：尝试 submodule 或 git rebase

---

## 13. 快捷速查（命令清单）

```bash
# 基础
git status
git add .
git commit -m "msg"
git pull
git push

# 远程
git remote -v
git remote set-url origin <url>

# 分支
git switch -c feature/x
git switch main
git merge feature/x

# 同步 fork
git remote add upstream <url>
git fetch upstream
git merge upstream/main
git push origin main

# SSH：查看本机 key、查看 agent、创建 key、测试 GitHub
ls -al ~/.ssh
ssh-add -l
ssh-keygen -t ed25519 -C "you@example.com"
ssh -T git@github.com
```