### 问题描述

用户在尝试通过 `apt purge` 命令卸载一个手动安装的 Linux 内核镜像时，收到如下错误：

```
E: 无法定位软件包 linux-image-unsigned-6.18.0-061800-generic6.18.0
E: 无法按照 glob ‘linux-image-unsigned-6.18.0-061800-generic6.18.0’ 找到任何软件包
```

通过 `dpkg --list | grep linux-image` 可见系统中存在以下内核镜像：

- `linux-image-6.14.0-36-generic`
- `linux-image-6.14.0-37-generic`
- `linux-image-generic-hwe-24.04`
- `linux-image-unsigned-6.18.0-061800-generic`

---

### 原因分析

错误原因是执行 `apt purge` 时输入的软件包名称不正确。  
实际安装的包名为：

```
linux-image-unsigned-6.18.0-061800-generic
```

但命令中误写为：

```
linux-image-unsigned-6.18.0-061800-generic6.18.0
```

即在包名末尾多加了版本号 `6.18.0`，导致 APT 无法识别该软件包。

---

### 解决方法

使用正确的包名执行卸载命令：

```bash
sudo apt purge linux-image-unsigned-6.18.0-061800-generic
```

若系统中还安装了对应的内核模块包（可通过 `dpkg --list | grep 6.18.0` 确认），建议一并卸载：

```bash
sudo apt purge linux-image-unsigned-6.18.0-061800-generic linux-modules-6.18.0-061800-generic
```

---

### 补充建议

对于通过官方仓库安装的旧内核，可使用以下命令自动清理不再需要的内核及其配置：

```bash
sudo apt autoremove --purge
```

注意：`autoremove` 不会移除手动安装的非官方内核（如本例中的 unsigned 6.18 内核），需手动指定包名进行清理。

---

### 验证操作结果

卸载完成后，可通过以下命令确认目标内核是否已移除：

```bash
dpkg --list | grep 6.18.0
```

若无输出，则表示相关包已成功清除。