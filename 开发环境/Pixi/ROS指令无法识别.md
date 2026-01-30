# 📝 技术笔记：在 Ubuntu 24.04 上通过 Pixi 运行 ROS 2 Humble

## 1. 核心原理：为什么选择 Pixi？

通常 ROS 2 的版本与 Ubuntu 发行版严格绑定（Humble 官方支持 22.04）。Ubuntu 24.04 由于系统库太新，无法直接通过 `apt` 安装 Humble。

* **隔离方案**：Pixi 利用 **Robostack** 生态（基于 Conda），将 ROS 2 的二进制文件及其所有旧版依赖库下载到项目本地的 `.pixi` 目录中。
* **非侵入性**：它不修改系统环境变量，不依赖系统的 Python 或编译器，实现了“原生性能”与“容器级隔离”的完美平衡。

---

## 2. 标准化的 `pixi.toml` 配置文件

经过多次调试，最专业且兼容性最好的 `pixi.toml` 结构应采用 **Workspace** 模式（Pixi 最新标准）：

```toml
[workspace]
name = "ros2-project"
version = "0.1.0"
# 优先级至关重要：robostack-humble 必须在 conda-forge 之前
channels = ["robostack-humble", "conda-forge"]
platforms = ["linux-64"]

[dependencies]
# ROS 2 核心：会自动拉取匹配的 Python 3.10
ros-humble-desktop = "*"
ros-humble-turtlesim = "*"

# 编译工具链：确保在隔离环境内进行构建
compilers = "*"
cmake = "*"
ninja = "*"
colcon-common-extensions = "*"

[tasks]
# 定义自动化工作流
build = "colcon build --symlink-install"
turtlesim = "ros2 run turtlesim turtlesim_node"
# 验证环境任务
check = "ros2 doctor"

```

---

## 3. 环境初始化与管理流程

| 步骤 | 命令 | 说明 |
| --- | --- | --- |
| **初始化** | `pixi init` | 创建项目骨架 |
| **同步依赖** | `pixi install` | 根据 `pixi.toml` 下载并创建隔离环境 |
| **进入环境** | `pixi shell` | 激活当前目录的 ROS 2 环境变量 |
| **执行构建** | `pixi run build` | 使用环境内的 `colcon` 进行编译 |

---

## 4. 关键故障排除（Lessons Learned）

## 🛑 问题 A：依赖冲突（Dependency Hell）

* **现象**：`pixi install` 报错，出现大量的 `ros2-distro-mutex` 冲突或 Python 版本不匹配。
* **原因**：手动指定了 `python = ">=3.11"` 或其他具体版本，干扰了 ROS 2 Humble 原生的 Python 3.10 依赖。
* **对策**：在 `[dependencies]` 中移除显式的 Python 版本限制，由 `ros-humble-desktop` 自动向上追溯。

## 🛑 问题 B：路径空格导致加载失败

* **现象**：`echo $ROS_DISTRO` 为空，或 `ros2` 命令找不到。
* **核心教训**：**Linux 下 ROS 2 项目路径严禁包含空格**（例如 `~/ROS 2/`）。空格会导致 Pixi 生成的环境激活脚本（Activation Scripts）在解析符号链接时发生截断。
* **修复**：重命名目录（如 `~/ROS2/`），删除旧的 `pixi.lock` 和 `.pixi/` 目录，重新执行 `pixi install`。

---

## 5. 验证环境是否成功的标准

在 `pixi shell` 中运行以下命令，全部通过则代表环境满血：

1. **环境变量**：`echo $ROS_DISTRO` 应该输出 `humble`。
2. **核心命令**：`ros2 doctor` 报告应显示网络和中间件正常。
3. **路径检查**：`which python` 应该指向项目下的 `.pixi/envs/default/bin/python`。
4. **图形化测试**：运行 `ros2 run rviz2 rviz2`，验证 GPU 加速和 QT 库是否兼容。

---

## 6. 专业建议

* **IDE 配置**：将 VS Code 的 Python 解释器手动指定为 `.pixi` 环境下的路径，可解决代码补全和语法报错。
* **混合开发**：在 `src` 下创建包时，保持 `CMakeLists.txt` 和 `package.xml` 的依赖与 `pixi.toml` 同步，确保 `colcon build` 能够正确链接。

---