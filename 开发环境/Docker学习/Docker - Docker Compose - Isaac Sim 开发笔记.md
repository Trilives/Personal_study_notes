# Docker / Docker Compose / Isaac Sim 开发笔记（对话整理）

日期：2025-12-16  
用户：Trilives

---

## 1. “通过名称启动”是什么意思？（Compose 常见三种“名称”）

### 1.1 按 **服务名** 启动（最常用）
`compose.yml` 中 `services:` 下的 key 就是服务名，例如 `api`、`redis`：

```bash
docker compose up -d api
docker compose up -d api redis
```

只启动该服务、不启动依赖（谨慎）：
```bash
docker compose up -d --no-deps api
```

### 1.2 按 **项目名（project name）** 启动（区分多套实例）
项目名影响容器/网络/卷的前缀：

```bash
docker compose -p vla1 up -d
docker compose -p vla1 ps
docker compose -p vla1 down
```

项目名来源优先级（常用）：
1. `docker compose -p xxx`
2. compose 顶层 `name: xxx`
3. 环境变量 `COMPOSE_PROJECT_NAME`
4. 默认：目录名

### 1.3 按 **容器名** 启动（不推荐作为主方案）
可以写死容器名：
```yaml
services:
  api:
    container_name: vla-api
```

但不推荐：难以多套实例、扩容 `--scale` 会受限。

---

## 2. docker compose vs 不用 compose 的区别

不仅是“有没有配置文件”，更是 **命令式 vs 声明式**：

### 2.1 不用 compose（docker run）
- 一条条 `docker run ...` 命令写参数（端口、卷、环境变量、网络等）
- 多容器依赖/网络/顺序需要自己管理
- 适合：单容器、临时测试

### 2.2 用 compose
- 用 `compose.yml` 声明一整套服务
- 一键 `up/down/logs/ps/exec`
- 默认创建网络，服务名可互相解析
- 更适合：多容器开发、团队协作、可复现环境

---

## 3. 如何把“自己的 Docker”做成稳定镜像（稳定=可复现）

核心：**Dockerfile + 锁定版本 + 自动构建发布**  
不推荐长期用 `docker commit`（只能救急快照）。

### 3.1 稳定镜像的关键原则
1. 固定 base image 版本（不要 `latest`，最好用 digest）
2. 固定依赖版本（requirements pin / lock 文件）
3. 构建尽量不依赖漂移的外部状态（避免 `apt-get upgrade`）
4. 模型权重通常别塞进镜像（建议挂载或外部下载）
5. Dockerfile 分层：先装依赖，再拷贝代码（提高缓存命中）
6. 健康检查 / 启动脚本（生产更需要）
7. multi-stage：构建与运行分离（可选）
8. CI：tag 触发 build & push（发布可追踪）

---

## 4. 日常开发：经常改代码，还要保存

推荐工作流：**环境稳定（镜像） + 代码频繁变（挂载） + 发布时再 build**

### 4.1 开发期（推荐）
- 依赖放进“环境镜像”
- 代码用 volume 挂载到容器里
- 服务用 reload/监控自动重启（uvicorn `--reload` 等）

### 4.2 发布期
把代码烘焙进镜像，打 tag，推送仓库：
```bash
docker build -t vla-infer:0.3.0 .
docker tag vla-infer:0.3.0 ghcr.io/<owner>/vla-infer:0.3.0
docker push ghcr.io/<owner>/vla-infer:0.3.0
```

---

## 5. Dockerfile 怎么写？为什么有 docker/ 文件夹？

### 5.1 Dockerfile 常见结构（骨架）
1. `FROM` 选基础镜像  
2. `ENV` 环境变量  
3. `WORKDIR`  
4. `RUN apt-get ...` 系统依赖  
5. `COPY requirements...`  
6. `RUN pip install ...`  
7. `COPY . .` 拷贝业务代码  
8. `CMD/ENTRYPOINT` 启动命令  

关键技巧：**变化少的步骤放前面，代码 COPY 放后面**。

### 5.2 为什么项目里有 `docker/` 目录？
常见原因：
- 多个 Dockerfile（dev/prod/cuda 版本矩阵）
- 放 entrypoint、启动脚本、nginx 配置等
- 多个 compose 文件（dev/prod/gpu）
- 更清晰的工程组织

---

## 6. Dockerfile 是“拉取镜像后要做的事”吗？

不是。  
Dockerfile 是构建镜像的配方：

- Dockerfile → `docker build` → Image（镜像）
- Image → `docker run`/compose → Container（容器实例）

拉取镜像只是下载别人做好的镜像：
```bash
docker pull ubuntu:22.04
```

---

## 7. Dockerfile 能自动产生吗？

“半自动”可行，“全自动且稳定”较难：

- 推荐：模板生成 + 补充项目入口、依赖、CUDA 需求
- 工具：`docker init` 可以生成基础骨架（但 GPU/CUDA 常需手改）
- 不推荐：从容器反推 Dockerfile 或只用 `docker commit`

---

## 8. Docker 构建镜像是不是把项目整个压缩进镜像？

不是“自动把整个目录压缩进镜像”。

- `docker build .` 会把 **build context**（目录）打包发给构建器
- 但哪些文件进入镜像取决于 Dockerfile 的 `COPY/ADD`
- 用 `.dockerignore` 排除大文件（模型、数据集、日志、.git 等）

---

## 9. 只有一个 compose.yml，它怎么知道容器名字并启动？

Compose 的单位是 **service**。容器名会自动生成：

```
<project>_<service>_<index>
```

- `service`：`services:` 下的 key，比如 `api`
- `index`：通常从 1 开始
- `project`：目录名或 `-p`/`name:` 指定

Compose 不是靠你手写容器名，而是用 project+service+labels 管理。

常用查看：
```bash
docker compose ps
```

---

## 10. `/bin/bash` vs `sleep infinity` 作为初始命令的区别

### 10.1 `/bin/bash`
- 更适合交互调试，需要 TTY/STDIN（`-it` 或 compose `tty: true`, `stdin_open: true`）
- 退出 bash 容器就停

### 10.2 `sleep infinity`
- 最稳定的“占位进程”，后台运行也不会退出
- 通过 `docker exec -it ... bash` 再进入容器

生产服务不要用这两者当最终命令，应运行真正服务进程。

---

## 11. Docker 中不同服务的区别：为什么一个服务构建的镜像另一个服务也在用？

Compose 的 service 只是“运行配置”。**镜像是全局共享的**。

常见原因：
1. 两个服务引用同一个 `image:` tag
2. 两个服务 `build:` 指向同一个 context（构建结果/层缓存复用）
3. 容器里能访问宿主机 docker.sock，在一个服务里 build 出来的镜像是宿主机全局镜像
4. 你以为共享镜像，其实共享的是 **volume**（数据目录）

---

## 12. 如何保存“进入容器后的操作”，下次继续？

要区分你保存的是什么：

### 12.1 保存数据/代码/模型（推荐）
用 bind mount/volume：
- 挂载代码目录（本地改立即生效）
- 挂载模型、缓存、输出目录（持久化）

### 12.2 保存安装的依赖/系统改动
- 应急：`docker commit <container> newimage:tag`
- 长期：把安装步骤写回 Dockerfile（可复现）

### 12.3 继续同一个容器（不删除容器）
```bash
docker start -ai <container>
# 或
docker start <container>
docker exec -it <container> bash
```

---

## 13. 删除一个“实例”怎么删？

### 13.1 普通 docker
```bash
docker ps -a
docker rm -f <container>
```

### 13.2 compose
删除某服务容器：
```bash
docker compose stop <service>
docker compose rm -f <service>
```

scale 多副本时可：
- 缩容：
```bash
docker compose up -d --scale api=1
```
- 或直接 `docker rm -f <具体容器名>`

整套删除：
```bash
docker compose down
docker compose down -v   # 慎用：会删卷数据
```

---

## 14. `network_mode: host` 能用主机代理吗？

可以，但要分清：

- host 网络只共享网络栈，不会自动继承主机代理配置
- 是否走代理取决于容器内是否设置 `HTTP_PROXY/HTTPS_PROXY/NO_PROXY` 或主机是否用透明代理

在 host 网络下，容器访问 `127.0.0.1:代理端口` = 主机本机代理端口（这是 bridge 模式做不到/不方便的点）。

---

## 15. Isaac Sim 构建时的错误：uv 安装失败原因与修复

用户构建日志关键报错：

- `curl: (7) Failed to connect to 127.0.0.1 port 7897 ...`
- `/sh: 1: cannot create /root/.bashrc: Permission denied`

### 15.1 原因 1：build 阶段代理地址用 `127.0.0.1:7897` 连不上
在 build 网络命名空间里 `127.0.0.1` 指向构建容器自身，不是宿主机。

解决思路：
- 构建时用 host 网络：`docker build --network=host ...`
- 或在 compose build 中加 `network: host`
- 或把代理地址改成宿主机可达地址（Linux 可用 host-gateway；Desktop 可用 `host.docker.internal`）

### 15.2 原因 2：基础镜像默认不是 root，写 `/root/.bashrc` 没权限
`nvcr.io/nvidia/isaac-sim:5.1.0` 可能默认用户不是 root。

解决：
- `USER root` 执行安装，再切回原用户
- 不要依赖 `.bashrc`，用 `ENV PATH=...` 更稳

建议把：
```dockerfile
echo 'export PATH="/root/.cargo/bin:$PATH"' >> /root/.bashrc
```
改为：
```dockerfile
ENV PATH="${HOME}/.cargo/bin:${PATH}"
```

---

## 16. 在 Isaac 环境下推荐使用 uv 吗？

结论：**看目的**。

- 追求稳定可复现（尤其生产镜像）：通常 `pip + 固定版本 + venv` 更省坑
- 追求开发效率/更快安装/依赖锁：可以用 uv，但要处理好：
  - 非 root 用户权限
  - 安装路径（`${HOME}/.cargo/bin`）
  - build 阶段网络/代理配置

---

## 附：本次对话中出现的错误截图

![image1](image1)

日志内容为 `isaac_sim_openpi` 的重复错误输出：
`[Error] [carb.progress.plugin] Asset event missing the AssetId event, ignoring`

---

## 下一步建议（可选）
如果要把 Isaac Sim + OpenPI 的 Dockerfile/compose 做到：
- 可复现构建
- 代理可用
- 非 root 权限不踩坑
- 开发时可挂载代码热更新

建议补充信息：
1. 你的代理软件与端口（如 7897）具体监听地址（127.0.0.1 还是 0.0.0.0）
2. Dockerfile 第 35-50 行（是否设置了 HTTP_PROXY/USER）
3. 目标：开发（挂载代码）还是发布（固定镜像）