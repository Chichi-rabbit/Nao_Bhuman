# SimRobot 使用指南

## 概述

SimRobot 是 BHuman 团队专为 RoboCup SPL（标准平台联赛）开发的机器人仿真平台。它支持在纯软件环境中运行完整的机器人控制代码，可用于步态测试、感知调试、策略验证，也可通过网络直接连接实体 NAO 机器人进行实时监控与调试。

**本文档主要介绍使用 SimRobot 进行代码调试的方法。**

---

## 关于代码来源与复现说明

本项目基于 [BHuman 开源框架](https://github.com/bhuman/BHumanCodeRelease) 进行开发，SimRobot 仿真器本身作为其子模块包含在框架内（位于 `Util/SimRobot/`，上游仓库：https://github.com/bhuman/SimRobot）。

我们在 BHuman 原始代码的基础上进行了针对性的算法修改与功能扩展，由于目前处于备赛阶段，这部分**修改后的代码暂不开源**。

如需复现本文档所描述的仿真调试流程，可直接使用 **BHuman 官方发布的原始代码**：

> 📦 **BHuman 官方代码仓库**：https://github.com/bhuman/BHumanCodeRelease

按照其仓库中的说明完成编译后，即可参照本文档操作 SimRobot。需要注意的是，由于我们的修改涉及机器人控制逻辑，部分仿真行为（如步态表现、感知策略等）可能与原始版本存在差异，但 SimRobot 的界面操作与调试流程本身是通用的，不受影响。

---

## 一、安装与编译

### 1.1 编译方式

以下三种方式均可编译 SimRobot，推荐根据工作习惯选择其中一种：

**方式一：CLion**  
在 CLion 右上角的运行配置下拉框中选择 `SimRobot`，直接点击编译并运行。

**方式二：命令行**  
在项目已完成基础编译的前提下，执行：
```bash
Make/Linux/compile SimRobot
```
编译产物位于 `Build/Linux/SimRobot/Develop/SimRobot`。

**方式三：脚本**  
使用项目提供的 `open.sh` 脚本一键完成编译与启动。

> ⚠️ **注意**：若修改了机器人控制代码后需要在仿真中验证，必须先重新编译整个项目，再启动 SimRobot，否则仿真运行的仍是旧版代码。

---

## 二、界面概览

SimRobot 启动后，界面主要由三个区域组成：左侧的 **Scene Graph（场景树）**、中央的 **视图区域**，以及 **Console（控制台）**。工具栏位于顶部，提供菜单和快捷操作。

### 2.1 工具栏菜单说明

| 菜单 | 功能 |
|------|------|
| **File** | 打开或关闭仿真场景文件（`.ros2d` / `.ros3`），场景文件通常位于 `Config/Scenes/` |
| **Edit** | 撤销、复制、粘贴等文本编辑操作 |
| **View** | 控制当前显示的视图面板及视图更新频率 |
| **Simulation** | 控制仿真的开始、暂停、重置与单步步进 |
| **B-Human** | 快速发送常用指令：点击站立图标让机器人起立，蹲下图标让机器人蹲下，移动图标允许手动控制机器人头部朝向 |
| **Scene** | 场景相关操作，包括鼠标拖拽模式、相机视角切换、渲染质量等 |
| **Add-ons** | 勾选 `File Editor` 可在软件内直接编辑场景文件与仿真脚本 |

### 2.2 场景树（Scene Graph）

场景树位于界面左侧，以树形结构显示当前仿真中的所有对象。打开场景文件后，双击树中的 `RoboCup` 节点可在中央区域打开三维场景视图。

若左侧未显示场景树，点击工具栏中类似 git 分支的图标即可打开。

### 2.3 视图区域

中央区域可同时显示多个视图面板，常用视图如下：

**3D 场景视图**  
双击场景树中的 `RoboCup` 打开。可实时观察机器人和球的位置，用鼠标拖拽可移动机器人或球。

**图像视图（Image Views）**  
显示机器人摄像头图像及其上的调试绘图，在图像坐标系中可视化感知结果：
```
# 以下机器人 Lower 摄像头图像为例
vi image lowerImage               # 创建图像视图
vid lowerImage representation:BallPercept:image    # 叠加球的感知结果
vid lowerImage representation:LinesPercept:image   # 叠加线段感知结果
```

**场地视图（Field Views）**  
在足球场坐标系中可视化机器人的定位与感知状态，适合全局调试：
```
vf worldState                     # 创建场地视图
vfd worldState fieldLines         # 叠加场地线
vfd worldState goalFrame          # 叠加球门
```

**行为视图（Behavior View）**  
显示当前所有活动的选项（Options）和技能（Skills）及其状态，是调试决策逻辑的核心工具。使用前需先发送调试请求：
```
dr representation:ActivationGraph
```

**关节数据视图（Joint Data View）**  
以表格形式显示所有关节的请求角度、传感器测量值、温度和负载：
```
dr representation:JointRequest
dr representation:JointSensorData
```

**模块视图（Module Views）**  
自动生成当前模块配置的有向图，由 Graphviz 的 dot 程序渲染。图中各元素含义：
- 黄色矩形：模块（Module）
- 蓝色椭圆：表示（Representation）
- 橙色虚线边框：从其他线程接收的表示
- 红色标签和边框：缺失的表示（配置错误）

**绘图视图（Plot Views）**  
实时绘制由代码中 `PLOT` 宏发送的数值数据，便于观察随时间变化的信号：
```
vp orientationX 200 -10 10                            # 创建绘图，显示最近200帧，范围-10到10
vpd orientationX representation:InertialData:angle:x  # 添加数据源
```

**时序视图（Timing Views）**  
显示各线程中所有计时器的最小/最大/平均运行时间及线程频率，用于性能分析：
```
dr timing
```

**数据视图（Data Views）**  
查看或修改机器人代码中通过 `MODIFY` 宏暴露的任意数据。视图有四种状态：
- **初始状态**：每秒 10 次自动从机器人同步数据
- **编辑状态**：停止同步以保留本地修改
- **修改状态**：视图标题末尾显示星号（`*`）
- **自动设置状态**：立即将修改值发送至机器人（标题末尾显示向上箭头 `↑`）

**色彩空间视图（Color Space Views）**  
在三维空间中可视化图像的颜色分布，支持 HSI、RGB、YCbCr 等色彩空间，适合调试颜色分类器。

**日志播放器视图（Log Player View）**  
控制日志文件的回放，快捷键如下：

| 按钮 | 快捷键 | 功能 |
|------|--------|------|
| ▶️ | Space | 开始播放 |
| ⏸️ | Space | 暂停播放 |
| ⏹️ | Home | 停止并回到开头 |
| 🔄 | End | 重复当前帧 |
| ◀️ | Left | 上一帧 |
| ▶️ | Right | 下一帧 |
| ⏮️ | Up | 跳转至上一个含图像的帧 |
| ⏭️ | Down | 跳转至下一个含图像的帧 |
| ⏪ | PageUp | 后退 100 帧 |
| ⏩ | PageDown | 前进 100 帧 |
| 🔁 | — | 循环播放 |

**注释视图（Annotation View）**  
显示日志文件中包含的所有注释，双击可跳转至对应帧。实时查看需发送：
```
dr annotation
```

### 2.4 控制台（Console）

底部控制台是与仿真交互的主要入口。所有调试命令均在此输入，详见[第四章调试命令参考](#四调试命令参考)。

---

## 三、仿真场景

场景文件位于 `Config/Scenes/`，扩展名为 `.ros3`（三维）或 `.ros2d`（二维）。不同场景适用于不同的调试场景，下面按用途分类介绍。

### 3.1 单机器人调试（BH 系列）

适合调试单个机器人的行为、步态和感知，资源占用较低，启动速度快。

| 场景文件 | 特点 |
|----------|------|
| `BH.ros3` / `BHFast.ros3` | 3D 场景，1 个活跃机器人 vs 5 个静止模型 |
| `BH2D.ros2d` | 2D 场景，视觉更简洁，适合快速验证逻辑 |
| `BHPerceptOracle.ros3` | 在机器人头顶显示编号，脚下显示定位和朝向，便于调试感知与定位 |

### 3.2 全场仿真（Game 系列）

模拟完整的 7v7 比赛，用于测试团队策略和跨机器人协作。

| 场景文件 | 特点 |
|----------|------|
| `Game.ros3` | 完整全场仿真，包含裁判系统（7v7） |
| `GameFast.ros3` | 不含裁判系统，启动更快，适合快速迭代测试 |
| `Game2D.ros2d` | 2D 全场仿真，便于俯视观察整体战术 |
| `GamePerceptOracle.ros3` | 叠加显示机器人内部感知状态，便于调试全场感知 |

### 3.3 己方策略测试（OneTeam 系列）

己方机器人正常运行，对方为静止模型，专注于验证己方策略而无需考虑对手动态。

| 场景文件 | 特点 |
|----------|------|
| `OneTeam.ros3` | 含裁判系统，7 活跃 vs 7 静止 |
| `OneTeamFast.ros3` | 不含裁判系统，资源占用更低 |
| `OneTeam2D.ros2d` | 2D 版本 |
| `OneTeamPerceptOracle.ros3` | 带感知状态叠加显示 |

### 3.4 特殊场景

| 场景文件 | 用途 |
|----------|------|
| `RemoteRobot.ros3` | 连接实体 NAO 机器人，实时监控其状态（详见[第五章](#五连接实体机器人)） |
| `ReplayRobot.ros3` | 加载并回放机器人日志文件，用于事后分析（机器人名为 "LOG"） |

### 3.5 仿真演示视频

> 📹 **仿真场景演示视频**   
> https://raw.githubusercontent.com/Chichi-rabbit/Nao_Bhuman/main/simrobot.mp4

---

## 四、调试命令参考

所有命令在底部 Console 中输入。`robot <name>` 命令用于选定目标机器人后，后续机器人命令将对该机器人生效。

### 4.1 全局命令

```bash
robot ? | all | <name>          # 选择目标机器人或机器人组
gc <state>                      # 设置 GameController 状态
ar {<feature>} (off | on)       # 启用/禁用自动裁判及其功能
call <file>                     # 执行脚本文件（批量发送命令）
ci off | on | <fps>             # 切换摄像头图像计算开关
st off | on                     # 切换时间仿真开关
```

### 4.2 连接命令

```bash
sc <name> <a.b.c.d>             # 连接到指定 IP 的实体机器人
sl <name> [(<file> | remote) [<scenario> [<location>]]]  # 回放日志或监控远程机器人
rc local <team_number> | remote <team_number> <broadcast>  # 启动远程控制连接
```

### 4.3 调试数据命令

```bash
dr ? [<pattern>] | off | <key> (off | on)      # 发送/取消调试请求
get ? [<pattern>] | <key> [? | save [<file>]]  # 显示或保存调试数据
set ? [<pattern>] | <key> (? | unchanged | <data>)  # 修改调试数据
poll                                            # 轮询所有可用调试请求和绘图
```

### 4.4 视图命令

```bash
vf <name>                       # 创建场地视图
vfd (all | <name>) <drawing> [on | off]  # 在场地视图中添加/移除调试绘图
vi <image> [<name>] [gain <value>]       # 创建图像视图
vid (all | <name>) <drawing> [on | off] # 在图像视图中添加/移除调试绘图
vp <name> <numOfValues> <min> <max>     # 创建绘图视图
vpd <name> <drawing> <color>            # 在绘图视图中添加数据曲线
```

### 4.5 仿真控制命令

```bash
mv <x> <y> <z> [<rotx> <roty> <rotz>]  # 移动选定的仿真机器人
mvb <x> <y> <z>                         # 移动球
pr <penalty>                            # 对仿真机器人施加处罚
sn [noiseType] [on | off]               # 开关传感器噪声仿真
si reset [<number>] | [number] [grayscale] [<file>]  # 保存机器人原始图像
```

### 4.6 模块与日志命令

```bash
mr <representation> (? | <module> | off | default)  # 切换表示的提供模块
log ...                                              # 记录和回放日志文件
kc hide | show | (press | release) <letter> <command>  # 绑定键盘快捷命令
```

### 4.7 自定义场景

若需调整仿真中的机器人数量、位置或配置，可编辑 `Config/Scenes/Includes/` 目录下对应的 `.rsi3` 文件，然后在场景文件（`.ros3`）中引用修改后的 include 文件。

---

## 五、连接实体机器人

SimRobot 支持通过网络连接实体 NAO 机器人，实时获取其传感器数据和运行状态，效果与仿真调试基本一致。

### 5.1 网络配置

连接前需关闭电脑的 DHCP，手动设置静态 IP：

| 连接方式 | 电脑 IP 网段 | 子网掩码 | 网关 |
|----------|-------------|----------|------|
| 无线（WiFi） | `10.0.1.x` | 255.255.0.0 | 10.0.0.1 |
| 有线（以太网） | `192.168.64.x` | 255.255.0.0 | 10.0.0.1 |

机器人本身的 IP 地址可在机器人胸口按钮处查看（单次按下胸口键，机器人会语音播报 IP）。

### 5.2 连接步骤

1. 在 SimRobot 中打开 `RemoteRobot.ros3` 场景。
2. 在 Console 中输入连接命令：
   ```bash
   sc Remote <机器人IP地址>
   # 例如：sc Remote 10.0.1.45
   ```
3. 连接成功后，场景树中的 Remote 机器人节点将变为活跃状态。

### 5.3 连接后可用的监控功能

连接实体机器人后，以下调试功能与仿真环境完全一致：

- **3D 场景视图**：双击 `RoboCup` 查看机器人在虚拟场地中的位置（基于机器人自身定位估计）
- **摄像头图像**：展开场景树中的 `robot.image`，可查看上下摄像头的原始图像和分割后图像
- **定位状态**：在场地视图的 `worldState` 绘图中查看机器人对自身在场上位置的估计
- **决策过程**：双击场景树中的 `robot.behaviour` 打开行为视图，实时观察决策树的执行状态
- **关节与传感器**：通过 `dr representation:JointSensorData` 查看实时关节数据

> 📹 实体机器人连接视频  
> https://raw.githubusercontent.com/Chichi-rabbit/Nao_Bhuman/main/%E5%AE%9E%E4%BD%93%E6%9C%BA%E5%99%A8%E4%BA%BA.mp4

---

## 六、常见问题

### 仿真卡住 / 无响应

**原因**：同时打开多个仿真场景占用内存过多，导致进程挂起。

**解决方法**：
```bash
top           # 查找占用资源异常的 SimRobot 进程，记录其 PID
kill <PID>    # 强制结束该进程
```

### 场景文件打开失败

1. 检查场景文件路径是否正确（应位于 `Config/Scenes/` 下）
2. 确认对应的 `.rsi3` include 文件存在且路径匹配
3. 确认当前使用的编译配置与场景所需一致

### 连接实体机器人失败

1. 确认电脑已关闭 DHCP 并设置了正确的静态 IP
2. 使用 `ping <机器人IP>` 验证网络连通性
3. 确认机器人上的 BHuman 代码已启动（机器人应处于站立或就绪状态）
4. 检查防火墙是否阻断了相关端口

---

## 附录：快速操作索引

| 目标 | 操作 |
|------|------|
| 打开场景 | File → 选择 `.ros3` 或 `.ros2d` 文件 |
| 打开 3D 视图 | 双击场景树中的 `RoboCup` |
| 让机器人起立 | B-Human 菜单 → 站立图标 |
| 选中目标机器人 | Console 输入 `robot <name>` |
| 发送调试请求 | Console 输入 `dr representation:XXX` |
| 移动机器人 | Console 输入 `mv <x> <y> <z>` |
| 移动球 | Console 输入 `mvb <x> <y> <z>` |
| 连接实体机器人 | 打开 `RemoteRobot.ros3`，输入 `sc Remote <IP>` |
| 结束卡住的进程 | 终端运行 `top` 找到 PID，再 `kill <PID>` |
