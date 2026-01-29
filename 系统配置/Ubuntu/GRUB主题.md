# 📝 Linux GRUB 高清主题安装与优化笔记

## 一、 环境准备

* **适用显示器**：4K (3840x2160) 及高分屏。
* **适用系统**：Ubuntu、Debian、Arch Linux、Fedora 等。
* **目标**：安装 Sleek Grub 主题并解决 4K 下的模糊问题。

---

## 二、 核心安装步骤

### 1. 下载并安装 Sleek Grub

从 GitHub 克隆项目并使用自动化脚本安装：

```bash
git clone https://github.com/sandesh236/sleek--themes.git
cd sleek--themes
sudo ./install.sh

```

*注意：安装路径通常为 `/usr/share/grub/themes/sleek/` 或 `/boot/grub/themes/sleek/`。*

### 2. 修正 4K 分辨率设置

防止 GRUB 默认回退到低分辨率导致拉伸模糊。

* 编辑配置文件：`sudo nano /etc/default/grub`
* 修改关键参数：
```text
# 强制指定 4K 分辨率，不建议使用 auto
GRUB_GFXMODE=3840x2160
# 保持内核启动时的分辨率一致
GRUB_GFXPAYLOAD_LINUX=keep
# 指定主题路径（根据实际安装位置微调）
GRUB_THEME="/usr/share/grub/themes/sleek/theme.txt"

```



---

## 三、 字体清晰度深度优化（进阶）

如果默认字体有锯齿，建议生成与原主题字号匹配的高清位图字体（`.pf2`）。

### 1. 生成匹配字号的高清字体

Sleek 主题标准字号通常为 18px 和 48px。使用 **JetBrains Mono** 渲染：

```bash
# 生成 48px 标题字
sudo grub-mkfont --output=/boot/grub/fonts/jb_48.pf2 --size=48 /usr/share/fonts/truetype/jetbrains-mono/JetBrainsMono-Bold.ttf

# 生成 18px 菜单字
sudo grub-mkfont --output=/boot/grub/fonts/jb_18.pf2 --size=18 /usr/share/fonts/truetype/jetbrains-mono/JetBrainsMono-Bold.ttf

```

### 2. 精准替换主题引用

编辑主题目录下的 `theme.txt`，将对应的 `font` 参数指向新生成的字体文件名（不带后缀）。

* **标题层**：`font = "jb_48"`
* **选项层**：`item_font = "jb_18"`

---

## 四、 常见问题与排错

### 1. 目录找不到（No such file or directory）

* 检查大小写：Linux 严格区分 `Sleek` 和 `sleek`。
* 查找真凶：使用 `grep "GRUB_THEME" /etc/default/grub` 确认系统当前加载的主题确切路径。

### 2. 界面排版乱掉（布局重叠）

* **原因**：自定义字体高度与原主题容器不匹配。
* **方案**：
1. 删除受损目录：`sudo rm -rf /usr/share/grub/themes/sleek/`
2. 重新运行 `install.sh` 脚本。
3. **核心建议**：在 4K 屏上，优先固定 `GRUB_GFXMODE=3840x2160`，通常即可获得足够清晰的效果，无需修改内部 `theme.txt`。



### 3. 应用更改

无论做了任何修改，必须运行以下命令才能生效：

* **Ubuntu/Debian**: `sudo update-grub`
* **Arch/Fedora**: `sudo grub-mkconfig -o /boot/grub/grub.cfg`

---

## 五、 常用维护命令汇总

* **查看字体列表**：`fc-list :file`
* **查看当前 GRUB 路径**：`ls -d /boot/grub*`
* **测试 GRUB 终端分辨率支持**（在 GRUB 界面按 C）：`videoinfo`

---