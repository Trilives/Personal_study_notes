# RustDesk 自建中继服务器部署与维护指南

## 一、目标

- 在阿里云 ECS 上部署 RustDesk 中继服务器（hbbs + hbbr）
- 固定服务器密钥（Key），确保客户端一次信任后永久可用
- 支持公网 IP 变更，无需反复清除客户端缓存
- 提供一键脚本，便于快速启动服务

---

## 二、核心原理

RustDesk 客户端通过公钥验证中继服务器身份。若服务器密钥变更，客户端会提示“Key 不匹配”。  
解决方案：**固定 `id_ed25519` 密钥对**，使其不随容器重建而改变。

只要密钥不变，即使 IP 更换，客户端在首次接受后将不再提示 Key 错误。

---

## 三、部署步骤

### 1. 清理旧环境（可选）

```bash
docker stop hbbs hbbr 2>/dev/null
docker rm hbbs hbbr 2>/dev/null
rm -rf ~/rustdesk-server
```

### 2. 创建项目目录

```bash
mkdir -p ~/rustdesk-server && cd ~/rustdesk-server
mkdir -p data/hbbs data/hbbr
```

### 3. 生成固定密钥对

```bash
docker run --rm \
  -v $PWD/data/hbbs:/root \
  ghcr.io/rustdesk/rustdesk-server:latest \
  hbbs -r 127.0.0.1:21117 &
sleep 3
kill %1 2>/dev/null || true
```

此操作会在 `data/hbbs/` 下生成 `id_ed25519` 和 `id_ed25519.pub`，后续所有部署将复用该密钥。

> 务必备份这两个文件，它们决定了服务器身份。

### 4. 配置安全组

在阿里云控制台为 ECS 实例的安全组添加入方向规则，开放以下端口：

- TCP: 21115, 21116, 21117, 21118, 21119
- UDP: 21116, 21117

来源建议设为 `0.0.0.0/0`（测试）或你的家庭公网 IP（生产）。

---

## 四、客户端配置

1. 打开 RustDesk → 设置 → “ID/中继服务器”
2. 填写：
    - ID 服务器：`<公网IP>:21116`
    - 中继服务器：`<公网IP>:21117`
    - Key：被控端留空，控制端输入上述生成的key
3. 首次连接时若提示“Key 不匹配”，点击“接受”或“信任”
4. 此后只要服务器密钥不变，即使 IP 更换，也不会再次提示

---

## 五、IP 变更处理流程

每次启动服务器时：

1. 获取当前公网 IP（从阿里云控制台或命令行）
2. 更新 `compose.yaml` 中的 IP 地址
3. 重启服务
4. 客户端仅需更新 IP 字段，无需重新信任

---

## 六、一键启动脚本（手动输入 IP 版）

创建脚本 `~/start-rustdesk.sh`：

```bash
#!/bin/bash

PROJECT_DIR="$HOME/rustdesk-server"
CONFIG_FILE="$PROJECT_DIR/compose.yaml"
HBBS_KEY_DIR="$PROJECT_DIR/data/hbbs"

echo "RustDesk 中继服务器启动工具"
read -p "请输入当前服务器的公网 IPv4 地址: " PUBLIC_IP

if ! [[ "$PUBLIC_IP" =~ ^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
    echo "输入的 IP 格式无效，请输入类似 47.98.123.45 的地址"
    exit 1
fi

echo "使用 IP: $PUBLIC_IP"

mkdir -p "$PROJECT_DIR"
cd "$PROJECT_DIR"
mkdir -p "$HBBS_KEY_DIR" "$PROJECT_DIR/data/hbbr"

cat > "$CONFIG_FILE" <<EOF
services:
  hbbs:
    image: ghcr.io/rustdesk/rustdesk-server:latest
    container_name: hbbs
    command: hbbs -r ${PUBLIC_IP}:21117
    volumes:
      - ./data/hbbs:/root
    restart: unless-stopped
    network_mode: host

  hbbr:
    image: ghcr.io/rustdesk/rustdesk-server:latest
    container_name: hbbr
    command: hbbr
    volumes:
      - ./data/hbbr:/root
    restart: unless-stopped
    network_mode: host
EOF

echo "compose.yaml 已更新"
docker compose down >/dev/null 2>&1 || true
docker compose up -d

sleep 3
if [ -f "$HBBS_KEY_DIR/id_ed25519.pub" ]; then
    PREFIX=$(cat "$HBBS_KEY_DIR/id_ed25519.pub" | cut -c1-8)
    echo "服务器公钥前8位: $PREFIX"
    echo "客户端首次连接时若提示 Key 不匹配，请核对是否为此值，并点击“接受”"
else
    echo "公钥尚未生成（首次运行可能需要几秒）"
    sleep 5
    if [ -f "$HBBS_KEY_DIR/id_ed25519.pub" ]; then
        PREFIX=$(cat "$HBBS_KEY_DIR/id_ed25519.pub" | cut -c1-8)
        echo "公钥已生成，前8位: $PREFIX"
    fi
fi

echo "RustDesk 服务已启动！"
echo "ID 服务器: $PUBLIC_IP:21116"
echo "中继服务器: $PUBLIC_IP:21117"
echo "（客户端设置中 Key 留空即可）"
```

### 使用方法

```bash
chmod +x ~/start-rustdesk.sh
~/start-rustdesk.sh
```

按提示输入当前公网 IP 即可。

---

## 七、常见问题处理

### 1. “Key 不匹配” 仍出现

- 确认 `data/hbbs/id_ed25519` 文件存在且未被删除
- 检查 `docker logs hbbs` 是否显示 “key pair found”
- 客户端首次连接时必须点击“接受”

### 2. Docker Compose 警告：多个配置文件

若同时存在 `compose.yaml` 和 `compose.yml`，删除多余文件：

```bash
rm -f ~/rustdesk-server/compose.yml
```

### 3. 脚本卡在获取 IP

因网络限制，自动获取 IP 可能失败。因此采用手动输入 IP 的脚本版本，避免依赖外部 API。

---

## 八、目录结构说明

```
~/rustdesk-server/
├── compose.yaml          # 服务配置文件（IP 可变）
├── data/
│   ├── hbbs/
│   │   ├── id_ed25519        # 私钥（固定，勿删）
│   │   └── id_ed25519.pub    # 公钥
│   └── hbbr/                 # hbbr 数据目录
```

只要保留 `data/hbbs/` 目录，服务器身份和设备 ID 将永久不变。

---

## 九、总结

- 密钥固定是解决“Key 不匹配”的关键
- 客户端只需首次接受服务器公钥
- IP 变更仅需更新配置并重启服务
- 使用手动输入 IP 的脚本，避免网络探测失败

此方案适用于个人自建、随用随停、IP 不固定的场景，兼顾安全性与易用性。