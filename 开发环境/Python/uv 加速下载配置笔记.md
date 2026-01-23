## 问题
uv 在国内下载速度慢

## 解决方案

### 方案一：使用代理（推荐）
如果主机已开启代理，配置环境变量让 uv 通过代理下载。

**根据你的代理设置，混合代理端口 7897 已启用，推荐使用。**

#### 设置步骤：

1. **编辑 bashrc 配置文件**
```bash
nano ~/.bashrc
```

2. **在文件末尾添加**
```bash
export HTTP_PROXY=http://127.0.0.1:7897
export HTTPS_PROXY=http://127.0.0.1:7897
export ALL_PROXY=http://127.0.0.1:7897
```

3. **重新加载配置**
```bash
source ~/.bashrc
```

4. **运行 uv sync**
```bash
GIT_LFS_SKIP_SMUDGE=1 uv sync
```

### 方案二：网络超时问题

如果下载时出现 `network timeout` 错误，增加超时时间：

```bash
export UV_HTTP_TIMEOUT=120
export HTTP_PROXY=http://127.0.0.1:7897
export HTTPS_PROXY=http://127.0.0.1:7897
export ALL_PROXY=http://127.0.0.1:7897
GIT_LFS_SKIP_SMUDGE=1 uv sync
```

或更激进的设置（300秒超时）：
```bash
export UV_HTTP_TIMEOUT=300
export HTTP_PROXY=http://127.0.0.1:7897
export HTTPS_PROXY=http://127.0.0.1:7897
export ALL_PROXY=http://127.0.0.1:7897
GIT_LFS_SKIP_SMUDGE=1 uv sync --timeout 300
```

### 方案三：使用国内 PyPI 镜像（备选）
如果代理有问题或不可用：
```bash
unset HTTP_PROXY
unset HTTPS_PROXY
unset ALL_PROXY
GIT_LFS_SKIP_SMUDGE=1 uv sync --index-url https://mirrors.aliyun.com/pypi/simple/
```

国内镜像源选项：
- 阿里云：`https://mirrors.aliyun.com/pypi/simple/`
- 清华：`https://pypi.tuna.tsinghua.edu.cn/simple/`
- 豆瓣：`https://pypi.douban.com/simple/`

## 排查步骤
1. **先试方案二**（增加超时时间）
2. **检查代理连接**：`curl -x http://127.0.0.1:7897 https://pypi.org/simple/numpy/ -I --connect-timeout 10`
3. **如果代理有问题，改用方案三**（国内镜像）

## 推荐顺序
1. **优先使用代理 + 增加超时**（最快最稳定）
2. **备选国内镜像**（代理不可用时使用）