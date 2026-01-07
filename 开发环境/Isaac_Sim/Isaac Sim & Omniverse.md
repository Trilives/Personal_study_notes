# 📚 Isaac Sim & Omniverse 全方位开发笔记 (Ubuntu 24.04 实践版)

## 一、 开发哲学：工作站与容器的混合流 (Hybrid Workflow)

在 Isaac Sim 开发中，不要试图一开始就在 Docker 中完成所有事情。最科学的流程是：

1. **工作站模式（Native GUI）**
    
    - **用途：** 场景搭建、USD 资产修改、物理参数可视化调试（红绿框）、初阶控制逻辑编写。
        
    - **优势：** **所见即所得**，无显示转发损耗，无按键劫持，调试效率极高。
        
2. **容器模式（Headless Docker）**
    
    - **用途：** 算法大规模训练（RL）、长期推理、版本隔离、云端部署。
        
    - **优势：** **环境一致性**，资源占用可控（无 UI 渲染开销）。
        

---

## 二、 基础环境配置 (Ubuntu 24.04 专供)

### 1. 显卡驱动与 Vulkan

Isaac Sim 严重依赖 Vulkan 接口，哪怕是在 Docker 中运行，宿主机的驱动也至关重要。

- **驱动检查**：执行 `nvidia-smi`，确保驱动版本 $\ge 535$。
    
- **基础库安装**（必须安装，否则 GUI 无法启动）：
    
    Bash
    
    ```
    sudo apt update && sudo apt install libvulkan1 vulkan-tools
    ```
    

### 2. 流式客户端 (Streaming Client) 适配

Ubuntu 24.04 移除了旧版 FUSE，且加强了 AppArmor 安全限制，这会导致下载的 `.AppImage` 客户端双击没反应或报错。

- **权限修复**：
    
    Bash
    
    ```
    chmod +x isaacsim-webrtc-streaming-client-*.AppImage
    ```
    
- **安装兼容库 (FUSE 2)**：
    
    Bash
    
    ```
    sudo apt install libfuse2t64
    ```
    
- **解除沙盒限制**（解决 `The SUID sandbox helper binary was found...` 报错）：
    
    - **临时解决**（推荐）：
        
        Bash
        
        ```
        ./isaacsim-streaming-client.AppImage --no-sandbox
        ```
        
    - **永久解决**（修改内核参数）：
        
        Bash
        
        ```
        sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0
        ```
        

---

## 三、 Docker 部署与持久化配置

### 1. 权限补丁 (UID 1234)

Isaac Sim 镜像内部固定使用用户 ID `1234` 运行。如果挂载的宿主机目录权限不对，容器会因为无法写缓存/日志而崩溃。

Bash

```
# 1. 创建必要的持久化目录结构
sudo mkdir -p ~/docker/isaac-sim/cache/main/ov \
             ~/docker/isaac-sim/cache/main/warp \
             ~/docker/isaac-sim/cache/computecache \
             ~/docker/isaac-sim/config \
             ~/docker/isaac-sim/data/Kit \
             ~/docker/isaac-sim/logs

# 2. 关键：将整个目录树的所有权递归交给 1234
sudo chown -R 1234:1234 ~/docker/isaac-sim
```

### 2. Docker Compose 核心逻辑

- **网络模式**：推荐使用 `network_mode: host`。
    
    - **原因**：WebRTC 串流需要大量且随机的 UDP 端口，Host 模式可以绕过复杂的端口映射，让容器直接使用宿主机 IP。
        
- **环境变量**：如果使用 Streaming Client，需确保 `OPENPI_SERVER_HOST` 配置正确（Host 模式下 `127.0.0.1` 通常可行，但在部分网络环境下需绑定局域网 IP）。
    

---

## 四、 核心难题：输入劫持与焦点管理

### 1. 问题现象

在流式客户端或 Headless 模式下，当你按下键盘（如 WASD）想控制机器人时，画面中的摄像机动了，但机器人没动。

- **原因**：Isaac Sim 的 **Viewport Editor（视口编辑器）** 默认抢占了输入焦点，将按键识别为“移动视角”。
    

### 2. 解决方案 A：隐藏 UI (最推荐)

在启动脚本时添加参数，强行关闭 UI 层。没有了 UI，按键自然会透传给底层的 Python 逻辑。

Bash

```
# 启动命令示例
./python.sh your_script.py \
    --ext-push omni.kit.livestream.webrtc \
    --/app/window/hideUi=true \
    --/gui/enabled=false
```

### 3. 解决方案 B：代码层拦截 (Python)

在 Python 脚本中通过 `omni.input` 注册具有最高优先级的回调，并返回 `True` 阻止事件向上传递。

Python

```
import omni.input as _input

def on_input_event(event):
    if event.type == _input.EventType.KEY_PRESS:
        if event.input == _input.KeyboardInput.W:
            # 执行你的逻辑
            move_robot_forward()
            # 关键：返回 True，告诉系统“这个按键我处理了，UI别想动”
            return True 
    return False

# 订阅输入事件，设置高优先级
_input.get_input_interface().subscribe_to_keyboard_events("Viewport", on_input_event)
```

---

## 五、 显示方案对比：为什么拒绝 X11 转发？

|**维度**|**X11 转发 (X11 Forwarding)**|**流式客户端 (Streaming Client)**|
|---|---|---|
|**性能**|极低（严重掉帧，Vulkan 支持极差）|**极高（GPU 硬件编码，延迟低）**|
|**稳定性**|易崩溃，依赖 X Server 环境|**极稳，专为容器开发设计**|
|**Ubuntu 24.04**|兼容性极差（因默认使用 Wayland）|**完美支持**|
|**适用场景**|**完全不推荐**|**Docker 可视化的唯一标准方案**|

---

## 六、 自动化运行脚本示例 (Template)

将此逻辑整合进你的 Python 脚本，以确保在 Docker 或工作站中能正常启动流式传输并加载 USD。

Python

```
import asyncio
import os
from isaacsim import SimulationApp

# 1. 启动仿真应用
# 如果在 Docker/服务器运行，必须设为 True；工作站调试可设为 False
config = {
    "headless": True, 
    "width": 1280, 
    "height": 720
}
kit = SimulationApp(config)

from omni.isaac.core.utils.stage import open_stage
from omni.isaac.core import SimulationContext

async def main():
    usd_path = "/workspace/data/scene.usd"
    
    # 2. 加载场景
    if open_stage(usd_path):
        print(f"Loading stage: {usd_path}")
        sim_context = SimulationContext()
        await sim_context.initialize_simulation_context_async()
        
        # 3. 核心：必须 Play，物理引擎和脚本才会运行
        # Headless 模式下没有界面按钮给你点，必须代码触发
        sim_context.play()
        
        print("Simulation is playing. Connect via Streaming Client...")
        
        while kit.is_running():
            # 4. 执行仿真步进
            # render=True 保证流式客户端能看到画面更新
            sim_context.step(render=True)
            
    kit.close()

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 七、 故障排查清单 (Troubleshooting)

1. **双击 AppImage 无反应/以镜像打开**：
    
    - 检查是否执行了 `chmod +x`。
        
    - 检查是否安装了 `libfuse2t64`。
        
2. **AppImage 报错 Sandbox**：
    
    - 使用 `--no-sandbox` 参数启动。
        
3. **Docker 启动报错 "Permission denied"**：
    
    - 宿主机对应目录是否属于 UID 1234？执行 `sudo chown -R 1234:1234 ~/docker/isaac-sim`。
        
4. **画面能看到，但机器人无法控制**：
    
    - 代码里有没有显式调用 `sim_context.play()`？
        
    - 是否开启了 `--ext-push omni.kit.livestream.core`？
        
    - 按键是否被 UI 劫持？尝试加 `--/app/window/hideUi=true`。
        
5. **Docker 容器直接崩溃退出**：
    
    - 检查脚本里是否设置了 `"headless": True`。如果是 False 且没有 X11 环境，必崩。
        
6. **OpenPI 连不上**：
    
    - 检查 Docker 是否使用了 Host 网络模式。
        
    - 检查防火墙 `sudo ufw status`。