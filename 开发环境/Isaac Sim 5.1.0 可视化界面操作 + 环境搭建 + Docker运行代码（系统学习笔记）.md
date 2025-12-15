> 目标：从“会启动”到“会用UI搭场景、做仿真、写脚本、在Docker里跑代码/调试”的完整学习路径。  
> 说明：以下以 Isaac Sim 5.1.0 为主，细节在不同发行方式（Launcher/standalone/Docker）会略有差异，但核心概念一致。

---

## 0. 学习路线建议（建议照顺序走）

1. **认识UI和基本概念**：Stage / USD / Prim / Layer / Physics Scene / World
2. **搭建最小可运行环境**：地面 + 光照 + 相机 + 物体 + 碰撞/刚体
3. **掌握常用面板与常用操作**：移动/旋转/缩放、对齐、测量、选中、复制、层管理
4. **仿真与物理**：重力、碰撞体、刚体、关节、时间步长、播放/暂停/重置
5. **传感器与数据**：Camera/Lidar/IMU，渲染设置，数据导出
6. **脚本化工作流**：Script Editor、Extension、独立Python脚本
7. **Docker开发与运行代码**：容器启动方式、启GUI/无GUI、挂载、执行脚本、远程调试

---

## 1. 核心概念速览（必须先理解）

### 1.1 USD / Stage / Prim
- **USD（Universal Scene Description）**：场景文件与场景图标准（.usd/.usda/.usdc）。
- **Stage**：当前打开的场景实例（“舞台”）。
- **Prim**：Stage中的节点（物体、灯、相机、Xform、Mesh等都是Prim）。
- **Xform**：可变换节点（位置/旋转/缩放），很多物体都是 Xform + Mesh 子节点结构。

### 1.2 Layer（层）与非破坏式编辑
- USD支持 **多层叠加**：同一个Prim可以被不同Layer“覆盖/增量修改”。
- 常见用途：把“资产原始文件”作为底层，把“本次布置/改参数”放在上层，便于复用与协作。

### 1.3 物理相关概念
- **Physics Scene**：物理世界配置（重力、时间步长、求解器等）。
- **Collider**：碰撞体（几何形状用于碰撞检测）。
- **Rigid Body**：刚体（受物理影响、会掉落/碰撞/受力）。
- **Joint**：关节（铰链、滑轨等）。
- **Default Ground Plane**：常见“地面”Prim（带碰撞体）。

### 1.4 仿真循环（Sim vs Render）
- 仿真通常以固定步长推进（physics dt），渲染以帧率刷新。
- 常见问题：**仿真太快/太慢、物体穿模、抖动**，往往与时间步、求解器迭代、碰撞体设置有关。

---

## 2. 可视化界面（UI）总览：你会经常用到的区域

> 不同布局略有差异，但常用面板基本固定。

### 2.1 Stage 面板（场景树）
- 位置通常在左侧：显示Prim层级。
- 你会用它做：
  - 查找Prim路径（例如 `/World/Robot`）
  - 选中/重命名/父子层级调整
  - 关注“哪个Prim在动/有组件”

常用操作：
- 右键：Add / Delete / Duplicate / Create Xform / Add Reference
- 搜索Prim名称
- 复制Prim路径（脚本常用）

### 2.2 Property 面板（属性）
- 位置常在右侧：选中Prim后，所有属性都在这里（transform、材质、物理组件、可见性等）。
- 常用：
  - 调整位置/旋转/缩放
  - 给物体添加刚体、碰撞体
  - 调整相机参数、灯光参数
  - 查看/设置自定义属性（Attributes）

### 2.3 Viewport（视口）
- 主要交互区域：鼠标导航、选择、操控Gizmo。

常见导航（大部分DCC类似）：
- 旋转视角：Alt + 左键拖拽
- 平移：Alt + 中键拖拽
- 缩放：Alt + 右键拖拽 或滚轮
- 聚焦到选中：F（非常重要）
- 多选：Shift / 框选（取决于设置）

### 2.4 Content/Asset Browser（资产浏览）
- 用于拖拽资产：环境、道具、机器人、材质等。
- 常见来源：
  - 本地资产目录
  - Nucleus（企业/团队资产）
  - Isaac Sim自带示例资产

### 2.5 Timeline / Playback（时间轴）
- Play / Pause / Stop / Reset
- 对“物理播放”控制非常关键：播放后刚体才会掉落/关节才会动。

### 2.6 Console / Script Editor（控制台/脚本）
- Console：看日志、报错。
- Script Editor：写Python片段直接操控Stage（建议尽早学）。

### 2.7 Extensions（扩展管理）
- Isaac Sim/Omniverse几乎所有能力都以 Extension形式提供。
- 常用：启用ROS2、传感器、Replicator、UI工具等。

---

## 3. 建立环境：从零搭一个可用的仿真场景（UI方式）

下面按“新建一个最小场景”步骤写，建议你亲手做一遍。

### 3.1 新建Stage
1. File -> New
2. 保存为一个 `.usd`（建议立即保存，避免资源引用路径混乱）

建议目录结构（项目化）：
- `project/`
  - `usd/`（场景）
  - `assets/`（自定义模型）
  - `scripts/`（脚本）
  - `logs/`

### 3.2 创建 /World 根节点（如果模板没带）
- Stage面板右键 -> Create -> Xform
- 命名 `World`（很多脚本默认 `/World`）

### 3.3 添加地面（Ground Plane）
常见方式之一：
- Create -> Physics -> Ground Plane（不同版本菜单名略不同）
- 确认它带 Collider（碰撞）属性

检查要点：
- 地面必须有 Collider，否则物体会穿过去。
- 地面位置通常在 z=0（或y=0，取决于坐标系设置）。

### 3.4 添加光照（Lighting）
至少放一个光，否则你看不到细节：
- Create -> Light -> Distant Light（常见“太阳光”）
- 或者 Dome Light（环境光）更适合室外/全局照明

建议：
- 先用 Dome Light 打底，再加 Distant Light 作为主光。
- 渲染变暗：提高光强、检查曝光设置。

### 3.5 添加一个可见物体（Cube/Sphere/Mesh）
- Create -> Mesh -> Cube
- 放在地面上方（例如 z=1）

### 3.6 给物体加物理属性（最关键）
目标：播放后它会掉落并落到地面。

在选中Cube后（Property面板）：
1. Add -> Physics -> Collider（或 “Add Collider”）
2. Add -> Physics -> Rigid Body（或 “Add Rigid Body”）
3. 确认：
   - Rigid Body: Enabled
   - Mass / Density 有合理数值
   - Collider shape 合理（Cube一般是 box）

### 3.7 添加 Physics Scene（若缺失）
很多模板会自动创建，否则需要手动建：
- Create -> Physics -> Physics Scene
- 设置：
  - Gravity：例如 (0, 0, -9.81)（注意轴向）
  - Time Step：先用默认
  - Solver：默认PhysX即可

### 3.8 播放仿真
- Timeline 点 Play
- 观察Cube下落并与地面碰撞停住
- Reset 回到初始状态（注意：某些状态可能需要“停止+重置”）

---

## 4. 常用功能清单（高频使用）

### 4.1 变换工具（Move/Rotate/Scale）
- 视口中 Gizmo 操作：拖轴移动、拖环旋转
- 精确数值输入：Property -> Transform

技巧：
- 对齐地面：把物体的Z设置到接近地面或用 snap 工具（若启用）
- 复制多个物体：Duplicate 或 Copy/Paste

### 4.2 坐标系与单位
- 关注：
  - Up Axis（Z-up 或 Y-up）
  - 单位（常见为米）
- 导入模型时要处理：缩放、旋转（例如从Blender/URDF来的模型方向可能不同）

### 4.3 资产引用（Reference）与实例化
> 一般不建议直接“导入并把几何写进场景”，更推荐 Reference 外部USD资产。

- Add Reference：把外部USD作为引用加载到当前Stage
- 好处：
  - 资产更新，场景自动跟随
  - 多场景复用
  - Layer更清晰

### 4.4 Layer管理
- 创建一个“修改层/布置层”（overlay layer）
- 把你的布局、物理参数修改放在上层
- 避免改坏资产原件

### 4.5 物理调参常见点
- 物体穿透/抖动：
  - 增加 solver iterations（位置/速度迭代）
  - 减小 time step
  - 确保碰撞体形状合适（convex/primitive）
- 质量、摩擦、弹性：
  - Mass/Density
  - Physics material（static/dynamic friction, restitution）

### 4.6 相机（Camera）与视角管理
- Create -> Camera
- 视口锁定到相机（Lock to Camera）
- 便于录制、生成数据、固定观察角度

### 4.7 传感器（常用）
- RGB Camera
- Depth Camera
- Lidar（2D/3D）
- IMU（惯导）

要点：
- 传感器通常挂在某个 Xform/机器人链接下
- 需要启用对应Extension
- 需要设置更新频率、分辨率、FOV等

### 4.8 渲染与显示模式
- Real-time 模式查看
- Path Tracing 模式更真实但更慢
- 调整：
  - 抗锯齿、曝光
  - 分辨率/帧率
  - 是否启用某些后处理

### 4.9 常用调试工具
- 显示碰撞体（Physics Debug）
- 显示坐标轴、网格、包围盒
- 通过Console看报错（Extension缺失、资源路径错误等）

---

## 5. UI方式建立“标准环境模板”（建议你固定复用）

做一个 `base_env.usd`，每次仿真都从它派生/引用：

建议包含：
- `/World`
- `/World/PhysicsScene`
- `/World/GroundPlane`
- `/World/Lights`（Dome + Distant）
- `/World/Cameras`（一个默认相机）
- 可选：几个markers / measurement工具

然后你的实际场景：
- `task_scene.usd` 引用 `base_env.usd`
- 在 overlay layer 加你的机器人、货架、障碍物等

---

## 6. 在 Isaac Sim 内运行代码：三种常见方式

### 6.1 Script Editor（最快上手）
适合：验证API、调试Prim路径、快速试验。

典型用法：
- 获取当前Stage
- 查找/创建Prim
- 修改Transform
- 播放/停止仿真

你会经常做：
- `print()`看路径
- 反复运行小段脚本

### 6.2 Extensions / Kit App（中级）
适合：做工具面板、可复用功能模块、团队共享。

思路：
- 创建一个自定义extension
- 提供UI + 逻辑
- 用配置文件启用

### 6.3 直接运行 Python 脚本（更工程化，Docker中常用）
适合：批量生成数据、自动化跑仿真、训练/评估。

方式：
- 用 Isaac Sim 的 python 启动器（在容器/本机路径会不同）
- 传入 `--/app/...` 或脚本参数
- 支持 headless（无GUI）跑仿真

---

## 7. Docker 启动后：如何运行 Isaac Sim、如何运行代码（重点）

> Docker模式通常有两类：
> 1) 你要 **带GUI** 远程看界面（通过WebRTC/Streaming 或 X11/VNC）
> 2) 你要 **无GUI** headless 跑仿真脚本（最常用于训练/数据生成/CI）

下面用“原则+常见命令模板”写；不同镜像的入口脚本不同，但思路一致。

---

## 7.1 容器内典型目录与入口
一般会有：
- Isaac Sim 安装目录（例如 `/isaac-sim` 或 `/opt/isaac-sim`）
- 一些启动脚本：
  - `isaac-sim.sh`（启动GUI app）
  - `python.sh`（用内置python环境跑脚本）
  - `omni.isaac.sim` 相关launcher

你应当先在容器里确认：
```bash
ls -la
ls -la /isaac-sim
find /isaac-sim -maxdepth 2 -name "*python*.sh" -o -name "*isaac*sim*.sh"
```

---

## 7.2 Docker里启动GUI（两种常见方法）

### A) WebRTC / Streaming（推荐，跨平台更稳）
思路：容器跑Isaac Sim，浏览器打开一个地址看渲染流。

要点：
- 容器要映射端口（例如 8211/8899等，具体看镜像文档）
- 启动参数里打开 streaming/webRTC 扩展

你需要确认：
- 你是否能在host访问容器端口
- GPU是否正确透传（`nvidia-smi` 容器内可用）

### B) X11 直连（Linux桌面方便）
思路：把host的X server socket挂进去，让容器直接弹窗。
要点：
- `-e DISPLAY`
- 挂载 `/tmp/.X11-unix`
- `xhost +local:docker`（注意安全）

---

## 7.3 Docker里运行脚本（headless / GUI均可）

### 7.3.1 推荐工作方式
- 你的代码在host：`./scripts`
- 运行容器时挂载：`-v $(pwd)/scripts:/workspace/scripts`
- 在容器内用 Isaac Sim 的 `python.sh` 跑：

典型命令结构：
```bash
/isaac-sim/python.sh /workspace/scripts/my_task.py --arg1 xxx
```

好处：
- 不污染镜像
- 迭代快
- 容器只当运行时

### 7.3.2 headless（无GUI）跑仿真
你需要在启动时或脚本里启用 headless：
- 启动参数：`--headless`
- 或 Kit 配置项（不同版本略不同）

常见形式（示意）：
```bash
/isaac-sim/python.sh /workspace/scripts/run_sim.py --headless
```

你要关注脚本里是否创建了渲染相关内容：  
headless下依然可以渲染（离屏渲染），但要确保开启对应渲染后端/设置合适的renderer。

### 7.3.3 GUI启动后，再在容器里跑脚本
两种方式：
1. **Script Editor里执行**（手动）
2. **另外开一个终端 exec 进容器**，运行 python 脚本（让脚本连接已有Kit会话通常更复杂，不同架构不同）

更通用的做法是：脚本自己启动一个Kit应用（即用python.sh启动），而不是去“附着到已有GUI进程”。

---

## 7.4 Docker开发调试建议

### 7.4.1 代码热更新
- 通过挂载目录实现“改完即生效”
- 如果是extension开发，也可以把extension目录挂载到 Isaac Sim 搜索路径

### 7.4.2 日志与缓存目录
建议把以下目录也挂载出来，避免容器内丢失：
- logs
- cache（shader cache、omniverse cache）

可以在容器内查：
- `~/.cache/` 相关目录
- Isaac Sim 的 log 输出路径（Console也能看到）

### 7.4.3 常见问题排查
- 容器内 `nvidia-smi` 无输出：GPU未透传（检查 `--gpus all`、nvidia-container-toolkit）
- GUI黑屏/卡死：多数是X11权限、驱动版本、OpenGL/Vulkan相关
- 资源引用丢失：挂载路径与USD引用路径不一致（建议使用相对路径或统一资产根目录）

---

## 8. 建议你建立的“最小可复用脚本模板”（学习用）

> 你没要求我贴具体API代码，但学习时建议你做一个 `scripts/00_sanity_check.py`，里面完成：
- 打印 Isaac Sim 版本、Python版本
- 启动/关闭仿真应用（如果是独立脚本模式）
- 创建新Stage
- 创建地面、立方体、加刚体与碰撞
- 跑100步仿真并输出物体位置
- 退出

再逐步新增：
- 加相机并离屏渲染保存图片
- 加关节与运动控制
- 加传感器并导出数据

---

## 9. 常用功能进阶（按应用场景整理）

### 9.1 机器人导入与控制（概览）
- URDF导入（如果你用ROS生态）
- USD机器人资产（NVIDIA提供的示例）
- Articulation（关节系统）：用于机械臂/移动底盘/人形

你最终会关注：
- 关节限制、驱动（position/velocity/effort）
- 控制频率与物理步长
- 传感器挂载（相机在头部/末端等）

### 9.2 合成数据（Replicator）
用途：生成标注数据（检测/分割/深度/姿态等）
- 随机化：光照、材质、位置、背景
- 生成COCO等格式
- headless批量跑最合适（Docker）

### 9.3 场景性能优化
- 模型面数过高、材质复杂会拖慢实时渲染
- 碰撞体用 primitive 或 convex 简化
- 控制传感器分辨率/频率
- 避免每帧过多Python操作（批量写属性、减少频繁查询）

---

## 10. 你可以照这个“练习清单”逐步掌握UI与脚本

### 练习1：搭建最小物理场景
- 地面 + 光 + cube刚体 + 播放下落

### 练习2：加入障碍物与材质
- 几个box当墙
- 改材质/颜色
- 测试摩擦系数

### 练习3：相机固定视角并导出截图
- 创建Camera
- 锁定视角
- 脚本离屏渲染保存png

### 练习4：导入机器人（URDF或USD）
- 放置到地面
- 确认关节可动
- 用脚本给关节一个简单正弦轨迹

### 练习5：Docker headless 批量跑仿真
- 参数化：不同初始位置跑N次
- 输出：每次落点/碰撞次数/截图

---

## 11. 我需要你补充的3个关键信息（我可以据此把“Docker启动+运行代码”写成完全可直接复制的命令）

你现在的描述是“在docker里启动了之后”，但不同镜像/启动方式命令差别很大。请回复下面信息，我就能把第7章改成“你的环境专用版”，并附上可直接运行的命令与脚本骨架：

1. 你用的镜像名与版本（例如 `nvcr.io/nvidia/isaac-sim:5.1.0` 之类）  
2. 你希望 **带GUI** 还是 **纯headless**（或两者都要）  
3. 你的宿主机系统（Ubuntu/Windows/WSL2）+ GPU型号 + 驱动大致版本

---

## 12. 附录：常用术语对照（方便查文档）
- Stage：场景
- Prim：场景节点/对象
- Xform：变换节点
- Layer：层
- Reference：引用外部USD资产
- Collider：碰撞体
- Rigid Body：刚体
- Articulation：关节链系统（机器人）
- Timeline：时间轴
- Extension：扩展模块
- Headless：无界面运行
- Offscreen Render：离屏渲染
- Nucleus：Omniverse资产服务器
