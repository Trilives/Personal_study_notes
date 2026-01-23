Ubuntu 是基于 Debian 的 Linux 发行版，使用 **Bash（或 Zsh 等）作为默认 shell**。所谓“Ubuntu 基本语句”，通常指的是在终端（Terminal）中常用的 **Linux 命令行操作命令**。以下是初学者最常用、最核心的 Ubuntu/ Linux 基础命令分类整理：


## 🖥️ 一、文件与目录操作

| 命令      | 说明        | 示例                                       |
| ------- | --------- | ---------------------------------------- |
| `ls`    | 列出目录内容    | `ls -l`（详细列表）、`ls -a`（显示隐藏文件）            |
| `cd`    | 切换目录      | `cd ~`（回家目录）、`cd ..`（返回上一级）              |
| `pwd`   | 显示当前路径    | `pwd`                                    |
| `mkdir` | 创建目录      | `mkdir my_folder`、`mkdir -p a/b/c`（递归创建） |
| `rmdir` | 删除**空**目录 | `rmdir empty_dir`                        |
| `rm`    | 删除文件或目录   | `rm file.txt`、`rm -r folder/`（删非空目录）⚠️   |
| `cp`    | 复制文件/目录   | `cp a.txt b.txt`、`cp -r dir1 dir2`       |
| `mv`    | 移动或重命名    | `mv old.txt new.txt`、`mv file ~/docs/`   |

> ⚠️ `rm` 删除不可逆！建议新手用 `rm -i`（确认删除）。

---

## 🔍 二、查看文件内容

| 命令 | 说明 | 示例 |
|------|------|------|
| `cat` | 显示整个文件 | `cat notes.txt` |
| `less` | 分页查看大文件（按 `q` 退出） | `less log.txt` |
| `head` | 查看文件开头（默认10行） | `head -n 5 file` |
| `tail` | 查看文件结尾（常用于日志） | `tail -f /var/log/syslog`（实时追踪） |

---

## 📦 三、软件包管理（Ubuntu 使用 `apt`）

| 命令 | 说明 |
|------|------|
| `sudo apt update` | 更新软件包列表（必先执行） |
| `sudo apt upgrade` | 升级已安装的软件 |
| `sudo apt install 软件名` | 安装软件（如 `sudo apt install git`） |
| `sudo apt remove 软件名` | 卸载软件（保留配置） |
| `sudo apt autoremove` | 清理无用依赖 |

---

## 🌐 四、网络相关

| 命令 | 说明 |
|------|------|
| `ping` | 测试网络连通性 | `ping google.com` |
| `wget` / `curl` | 下载文件 | `wget https://example.com/file.zip` |
| `ip a` 或 `ifconfig` | 查看 IP 地址（`ifconfig` 需安装 `net-tools`） |

---

## 🔐 五、用户与权限

| 命令 | 说明 |
|------|------|
| `whoami` | 显示当前用户名 |
| `sudo` | 以管理员权限运行命令 | `sudo apt update` |
| `chmod` | 修改文件权限 | `chmod +x script.sh`（添加可执行权限） |
| `chown` | 修改文件所有者 | `sudo chown user:group file` |

---

## 🧰 六、系统信息与管理

| 命令 | 说明 |
|------|------|
| `uname -a` | 查看内核版本 |
| `df -h` | 查看磁盘空间（人类可读） |
| `free -h` | 查看内存使用 |
| `top` / `htop` | 查看进程（`htop` 更友好，需安装） |
| `shutdown` / `reboot` | 关机 / 重启 | `sudo reboot` |

---

## 📁 七、路径与特殊符号

- `~`：代表当前用户的家目录（如 `/home/vla`）
- `/`：根目录
- `.`：当前目录
- `..`：上一级目录
- `/mnt/c/`：WSL 中访问 Windows C 盘的路径

---

## 💡 小技巧

- 按 **Tab 键** 自动补全文件名或命令
- 按 **↑ ↓ 方向键** 查看历史命令
- `man 命令名`：查看命令帮助手册（如 `man ls`）
- `command --help`：快速查看用法（如 `ls --help`）

---

## ✅ 新手推荐工作流（WSL 示例）

```bash
# 1. 更新系统
sudo apt update && sudo apt upgrade -y

# 2. 创建项目目录
mkdir ~/projects
cd ~/projects

# 3. 从 Windows 复制文件（假设在桌面）
cp /mnt/c/Users/你的名字/Desktop/code.zip ./

# 4. 解压并进入
unzip code.zip
cd my_app

# 5. 安装依赖并运行（以 Python 为例）
sudo apt install python3
python3 app.py
```

---

这些命令不仅适用于 Ubuntu，也适用于大多数 Linux 发行版（如 Debian、CentOS 等略有差异）。掌握它们，你就已经能高效使用 WSL 或 Ubuntu 终端了！