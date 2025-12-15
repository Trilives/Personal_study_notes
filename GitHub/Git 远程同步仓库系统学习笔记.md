## 前提条件

### 1. 安装 Git

- **Windows**：  
    下载并安装 [Git for Windows](https://git-scm.com/download/win)。安装完成后可使用 Git Bash 或在 PowerShell/命令提示符中使用 Git。
    
- **Ubuntu / Debian 系 Linux**：  
    打开终端，运行以下命令：
    
    ```bash
    sudo apt update
    sudo apt install git
    ```
    
- **其他 Linux 发行版**：  
    使用对应包管理器安装 Git（如 `yum install git`、`dnf install git`、`pacman -S git` 等）。
    

### 2. 配置 Git 用户信息（首次使用需设置）

在任意终端中执行以下命令（只需一次）：

```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

请将 `"Your Name"` 和 `"your.email@example.com"` 替换为你在 GitHub 上注册的真实姓名和邮箱。

### 3. 创建远程 GitHub 仓库

1. 登录 [GitHub](https://github.com/)
2. 点击页面右上角的 `+` 号，选择 **New repository**
3. 填写仓库名称（Repository name），例如 `my-project`
4. **重要**：不要勾选 “Add a README file”、“Add .gitignore” 或 “Choose a license”，以免与本地文件冲突
5. 点击 **Create repository**
6. 复制生成的仓库 URL（HTTPS 格式，如 `https://github.com/username/my-project.git`）

---

## 上传本地文件夹到 GitHub 的完整步骤

### 步骤 1：进入本地项目目录

打开终端（Windows 使用 Git Bash、PowerShell 或 CMD；Ubuntu 使用 Terminal），切换到你的本地文件夹路径：

- **Linux / macOS 示例**：
    
    ```bash
    cd ~/Documents/my-project
    ```
    
- **Windows 示例（Git Bash）**：
    
    ```bash
    cd /c/Users/YourName/Documents/my-project
    ```
    
    或（PowerShell）：
    
    ```powershell
    cd C:\Users\YourName\Documents\my-project
    ```
    

### 步骤 2：初始化本地 Git 仓库（如果尚未初始化）

```bash
git init
```

此命令会在当前目录创建一个隐藏的 `.git` 文件夹，用于跟踪版本。

### 步骤 3：将所有文件添加到暂存区

```bash
git add .
```

说明：

- `.` 表示添加当前目录及其子目录下所有未被忽略的文件
- 若只想添加特定文件，可替换为具体路径，如 `git add file.txt`

> 建议：在 `git add .` 之前，可先创建 `.gitignore` 文件，排除不需要上传的内容（如编译产物、日志、虚拟环境等）。

### 步骤 4：提交更改

```bash
git commit -m "Initial commit"
```

该命令将暂存区的更改保存为一个本地提交，并附带提交信息。

### 步骤 5：关联远程 GitHub 仓库

```bash
git remote add origin https://github.com/username/my-project.git
```

将 `https://github.com/username/my-project.git` 替换为你在 GitHub 上创建的仓库地址。

> 可通过 `git remote -v` 验证是否成功添加远程仓库。

### 步骤 6：推送代码到 GitHub

```bash
git push -u origin main
```

说明：

- `-u` 参数将本地 `main` 分支与远程 `origin/main` 关联，后续只需 `git push` 即可
- 如果你的仓库默认分支是 `master`（旧版 GitHub 项目），则使用：
    
    ```bash
    git push -u origin master
    ```
    

> 如何确认默认分支名？  
> 在 GitHub 仓库页面顶部，分支下拉菜单中高亮显示的即为默认分支（现代新仓库通常为 `main`）。

---

## 身份认证方式（关键步骤）

GitHub 自 2021 年 8 月起**不再支持使用账户密码进行 Git 操作认证**。必须使用以下两种方式之一：

### 方式一：使用 Personal Access Token（推荐用于 HTTPS）

1. 生成 Token：
    
    - 登录 GitHub → 点击头像 → **Settings**
    - 左侧菜单选择 **Developer settings** → **Personal access tokens** → **Tokens (classic)**
    - 点击 **Generate new token** → **Generate new token (classic)**
    - 填写 Note（如 "My PC Git"），设置过期时间
    - 在权限部分勾选 **repo**（至少包含此权限）
    - 滚动到底部点击 **Generate token**
    - **立即复制并安全保存该 Token（只显示一次！）**
2. 推送时认证：
    
    - 当执行 `git push` 时，系统会弹出登录窗口（Windows 凭据管理器）或在终端提示输入用户名和密码
    - **用户名**：你的 GitHub 用户名
    - **密码**：粘贴刚才生成的 Personal Access Token（不是你的 GitHub 登录密码！）

> 后续操作中，凭据可能被系统缓存，无需重复输入。

### 方式二：使用 SSH 密钥（更安全，适合频繁开发者）

1. 生成 SSH 密钥（若尚未生成）：
    
    ```bash
    ssh-keygen -t ed25519 -C "your.email@example.com"
    ```
    
    - 按回车使用默认路径（`~/.ssh/id_ed25519`）
    - 可设置密码（passphrase）以增强安全性（可选）
2. 启动 ssh-agent 并添加密钥（Linux / macOS）：
    
    ```bash
    eval "$(ssh-agent -s)"
    ssh-add ~/.ssh/id_ed25519
    ```
    
3. 将公钥添加到 GitHub：
    
    - 查看公钥内容：
        
        ```bash
        cat ~/.ssh/id_ed25519.pub
        ```
        
    - 复制全部输出内容（以 `ssh-ed25519 ...` 开头）
    - GitHub → Settings → **SSH and GPG keys** → **New SSH key**
    - Title 填写设备名称（如 "My Ubuntu Laptop"），Key 粘贴公钥内容，点击 **Add SSH key**
4. 修改远程仓库地址为 SSH 格式：
    
    ```bash
    git remote set-url origin git@github.com:username/my-project.git
    ```
    
5. 推送代码：
    
    ```bash
    git push -u origin main
    ```
    
    此时无需输入用户名或密码（除非设置了 passphrase）。
    

---

## 常见问题与注意事项

### 1. 推送失败：`fatal: The current branch main has no upstream branch`

说明未设置上游分支。解决方法：

```bash
git push --set-upstream origin main
```

### 2. 推送失败：`remote: Permission denied`

- 检查仓库 URL 是否正确（是否属于你的账号或你有写权限）
- 检查认证方式（Token 是否有效、SSH 公钥是否添加）

### 3. 本地已有 Git 仓库？

若文件夹已存在 `.git` 目录，先检查远程是否已配置：

```bash
git remote -v
```

如已存在错误的远程地址，可删除后重新添加：

```bash
git remote remove origin
git remote add origin <your-new-url>
```

### 4. 忽略不需要上传的文件

在项目根目录创建 `.gitignore` 文件，例如：

```gitignore
# Python
__pycache__/
*.pyc
venv/

# Node.js
node_modules/

# Logs
*.log

# OS generated
.DS_Store
Thumbs.db
```

### 5. 更改默认分支名（可选）

若希望本地默认分支名为 `main`（而非 `master`）：

```bash
git config --global init.defaultBranch main
```

此设置仅影响未来新建的仓库。

---

## 完整命令示例（适用于 Git Bash / Ubuntu Terminal）

```bash
# 进入项目目录
cd /path/to/your/project

# 初始化仓库
git init

# 添加所有文件
git add .

# 提交
git commit -m "Initial commit"

# 关联远程仓库（HTTPS）
git remote add origin https://github.com/your-username/your-repo.git

# 推送到 GitHub（首次需认证）
git push -u origin main
```

---

完成以上步骤后，你的本地文件夹内容将成功上传至 GitHub 仓库。后续修改只需执行 `git add`、`git commit`、`git push` 即可同步更新。