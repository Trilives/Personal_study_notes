# 现代 Python 项目开发笔记 (基于 uv + src 布局)

## 1. 推荐的项目结构：src 布局

目前 Python 社区最推荐使用的是 **src 布局**。这种结构将源码放入 `src` 目录，能有效避免测试时误导入本地代码的问题（Import Confusion），并保持根目录整洁。

**目录树示例：**

```text
my_project/
├── .python-version    # uv 生成的 Python 版本锁定文件
├── pyproject.toml     # 核心配置文件（依赖、构建、元数据）
├── uv.lock            # 依赖版本锁定文件（确保环境一致性）
├── README.md          # 项目说明
├── LICENSE            # 授权协议文件
├── .gitignore         # Git 忽略配置
├── .venv/             # uv 自动创建的虚拟环境目录
├── src/               # 源码根目录
│   └── my_package/    # 实际的包
│       ├── __init__.py
│       └── main.py
├── tests/             # 测试代码目录
│   └── test_main.py
└── data/              # (可选) 本地数据或资源

```

## 2. 核心配置：pyproject.toml (配合 uv)

`pyproject.toml` 取代了传统的 `setup.py` 和 `requirements.txt`。在使用 `uv` 时，底层通常使用 `hatchling` 作为构建后端。

**配置模板：**

```toml
[project]
name = "my_project"
version = "0.1.0"
description = "项目描述"
readme = "README.md"
requires-python = ">=3.12"
# 作者信息
authors = [
    { name = "你的名字", email = "your_email@example.com" }
]
# 许可证引用
license = { file = "LICENSE" }
# 项目依赖（由 uv add 自动管理）
dependencies = [
    "requests>=2.31.0",
]
# 分类标签（用于 PyPI）
classifiers = [
    "Programming Language :: Python :: 3",
    "License :: OSI Approved :: MIT License",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

# 关键配置：显式指定源码路径
[tool.hatch.build.targets.wheel]
packages = ["src/my_package"]

# uv 专用的开发依赖配置
[tool.uv]
dev-dependencies = [
    "pytest>=8.0.0",
]

# 命令行入口配置
[project.scripts]
my-app = "my_package.main:app"

```

## 3. 关键概念辨析

### 3.1 pyproject.toml 与 __init__.py 的区别

* **pyproject.toml (构建层面/物流单)**
* **受众**：构建工具 (uv, pip, build)。
* **作用**：告诉工具去哪里打包代码（如 `src` 目录），需要什么依赖。它是代码安装前的物理路径指引。


* **__init__.py (运行层面/说明书)**
* **受众**：Python 解释器。
* **作用**：将文件夹标识为 Python 包。当执行 `import` 时首先运行此文件。它是代码安装后的逻辑入口。



### 3.2 .python-version 与 requires-python 的区别

* **requires-python (在 pyproject.toml 中)**
* **含义**：兼容性范围。
* **示例**：`>=3.12`。
* **作用**：告诉用户和安装程序，这个库最低支持到哪个版本。


* **python-version (独立文件)**
* **含义**：具体的开发版本锁定。
* **示例**：`3.12.4`。
* **作用**：告诉 `uv` 在本地开发环境中使用哪个具体的解释器。`uv` 会自动下载并配置此版本，确保团队开发环境完全一致。



## 4. Python 版本选择策略

* **最佳策略**：追求“次新版” (Latest - 1)。
* **推荐版本**：Python 3.12 (截至目前)。
* 理由：性能优秀（Faster CPython），且生态兼容性好。


* **最新版 (如 3.13)**：适合尝鲜，但可能面临第三方库（如 PyTorch, Pandas）未适配的问题，特别是涉及 C 扩展的库。
* **旧版本 (3.8及以下)**：面临停止维护 (EOL) 风险，应尽快迁移。

## 5. 开源许可证 (License) 选择

在 GitHub 添加许可证时，根据使用限制从宽到严推荐：

1. **MIT License (推荐)**
* **特点**：极度宽松。
* **权利**：用户可以任意使用、修改、闭源商用。
* **义务**：仅需保留原作者版权声明。
* **适用**：希望尽可能广泛传播的开源项目。


2. **Apache License 2.0**
* **特点**：宽松且规范。
* **权利**：同 MIT，允许商用。
* **额外**：明确授予专利权，防止专利流氓行为。
* **适用**：正规的商业开源项目或大型框架。


3. **GNU GPL v3.0**
* **特点**：强传染性（Copyleft）。
* **限制**：如果使用了该代码，衍生项目也必须开源（不能闭源商用）。
* **适用**：强烈的开源理念捍卫者，防止代码被大公司私有化。


4. **The Unlicense**
* **特点**：放弃所有权利，贡献给公共领域。



**操作提示**：在 GitHub 创建文件时输入 `LICENSE` 可直接选择模板生成。生成后需在 `pyproject.toml` 中添加 `license = { file = "LICENSE" }`。

## 6. uv 常用命令速查

* **初始化项目**：`uv init --lib my_project`
* **添加依赖**：`uv add requests`
* **添加开发依赖**：`uv add --dev pytest`
* **同步环境**（根据配置文件安装所有依赖）：`uv sync`
* **运行脚本/命令**（自动处理路径）：`uv run python -m my_package.main` 或 `uv run my-app`
* **锁定 Python 版本**：`uv python pin 3.12`