# 前言：项目背景与价值

想象一下你戴上 VR 头显，用手柄抓取虚拟物体，现实中的机械臂同步完成同样的动作——这就是 TeleGrip 的核心。
 本文将带你从源码角度理解它是如何实现“虚拟到现实”的信号映射与控制闭环的。

<img src="./assets/image-20251010095248617.png" alt="image-20251010095248617" style="zoom:67%;" />

GitHub链接：https://github.com/DipFlip/telegrip

对应deepwiki：https://deepwiki.com/DipFlip/telegrip

# 一、环境配置与运行指南

关于该部分在项目的 GitHub 界面已经写的非常详细，此处仅作简略总结，环境配置如下表所示：

| 项目类别                 | 内容说明                    | 示例命令 / 备注                                              |
| ------------------------ | --------------------------- | ------------------------------------------------------------ |
| **硬件要求**             | 机器人硬件                  | 一台或两台 **SO100 机械臂**（USB 串口连接）                  |
| **软件环境**             | Python 环境                 | Python ≥ 3.8，并安装相关依赖包                               |
| **VR 设备（可选）**      | VR 头显                     | Meta Quest / 其他支持 WebXR 的设备（无需额外安装应用）       |
| **基础依赖项目**         | 安装 LeRobot（基础功能库）  | `网址：https://github.com/huggingface/lerobot.git`           |
| **TeleGrip 主程序**      | 安装 TeleGrip（VR 控制端）  | `网址：https://github.com/DipFlip/telegrip.git`              |
| **证书配置**             | SSL 证书自动生成            | 若未检测到 `cert.pem` 与 `key.pem`，系统会自动创建           |
| **手动生成证书（可选）** | 使用 OpenSSL 生成自签名证书 | `openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -sha256 -days 365 -nodes -subj "/C=US/ST=Test/L=Test/O=Test/OU=Test/CN=localhost` |
| **运行验证**             | 启动 TeleGrip 测试服务      | 在 TeleGrip 根目录运行示例脚本，确认 VR 与机械臂通信正常     |

项目是在 Lerobot 的基础上进行的**二次开发**，使用硬件为典型 6 自由度入门机械臂 SO100，在各个平台均有销售，一套主从臂价格在1400 左右，但注意项目视频中为两从臂；

VR 头显为低成本的 Meta Quest，主要用于采集手柄的位姿信息，注意此处采集采用的是 VR 端的浏览器插件 A-FRAME，对硬件本身要求不高，如无特别需要，最小内存的 **Meta Quest 3S** 完全够用，其价格在 2000 左右；

项目配置环境时首先要配置 SO100 机械臂的驱动，表中已经给出项目地址，其中包含详细的配置流程，包括 conda 建立新环境、pip 一键安装等，详细如 conda 网络配置等不做概述，此处要**注意的点**为，在执行完 `pip install -e .`  后，要再运行 `pip install 'lerobot[feetech]'`，才能正常运行项目；还有关于 Weights and Biases 的注册等等，不在此处做过多概述；安装完成后可执行example 1 进行验证;

此处完成环境配置并将机械臂通过 USB 连接到电脑后，需要执行 `python -m lerobot.find_port`  来寻找**机械臂的端口号**，详细在机械臂的相关二次开发教程中有详细说明。

SO100 安装完成后，开始进行本项目的配置，可在之前 conda 创建的环境中接着进行配置，在 git clone 之后直接 `pip install -e .`即可，中间可能会报错停止，重复几次即可；此处要注意的点在于 OpenSSL 生成签名证书，如果直接没有进行过相关配置，是**必须要进行执行的**，最好在项目目录执行该命令，后续在运行时，证书在哪个目录下，就要在哪个目录进行运行。

首次运行时需要进行**机械臂活动范围的标定**，如果之前没有在 LeRobot 进行过，是必要的步骤，标定需要在配置的环境下执行如下命令 `telegrip --log-level info --left-port` 机械臂的端口号，此处可在 LeRobot 的官方教程或二次开发文件中查找 机械中值的位置并自己活动各个电机的活动范围，标定完成之后，打断进行正常的运行，得到网址在本机的浏览器运行连接机械臂，并在 VR 端的浏览器输入网址，即可进行遥操；只要机械臂连接成功，即可通过键盘进行调试。

特别注意，在 Ubuntu 系统部署的用户在执行时需要提前给定 机械臂端口**权限**，如：`sudo chmod 666 /dev/ttySO100*`

# 二、项目流程与源码解析

在 GitHub 上有简略的运行流程图如下：

<img src="./assets/image-20251010111534087.png" alt="image-20251010111534087" style="zoom: 67%;" />

由图可知，输入数据包括 VR 手柄与键盘，前者通过 WebSocket 进行数据传输，后者监听运行服务器的键盘，接着一块送入Control Loop 中进行解算处理，最后将解算的位姿更新到 pybullet 与实际硬件。其与代码对应关系为如下：

| 架构图模块               | 对应代码文件/目录                                            | 功能说明                                                     |
| ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **VR Controllers**       | `web-ui/vr_app.js`、`index.html`                             | 负责从 VR 设备（如 Meta Quest）采集手柄位姿数据（位置 + 姿态）并通过 WebSocket 发送到服务器。 |
| **Keyboard**             | `telegrip/inputs/keyboard_listener.py`（在 `inputs` 目录中） | 监听键盘输入事件（例如切换模式、重置姿态等），作为替代或辅助控制输入。 |
| **WebSocket Server**     | 由 `web-ui/vr_app.js` 启动                                   | 建立 WebSocket 通信，接收来自 VR 前端的实时控制指令流。      |
| **Keyboard Listener**    | `telegrip/inputs/keyboard_listener.py`                       | 接收键盘事件，将其转换为标准化的控制指令放入命令队列。       |
| **Command Queue**        | `telegrip\inputs\vr_ws_server.py`                            | 负责接收来自不同输入源（VR / 键盘）的指令并排队，提供线程安全的统一接口供主控制循环读取。 |
| **Control Loop**         | `telegrip/control_loop.py`                                   | TeleGrip 的核心控制逻辑：从命令队列中读取最新的控制数据，进行插值（`interpolation.py`）、限幅与更新，然后分发到机器人或仿真模块。 |
| **Robot Interface**      | `telegrip/core/robot_interface.py`                           | 封装与实际机械臂（SO100）的通信逻辑：建立串口连接、发送关节角、读取状态、执行运动控制命令。 |
| **SO100 Robot Hardware** | `/URDF/SO100/`                                               | 定义 SO100 机械臂的 URDF 模型和运动学结构，用于与实际硬件对齐或仿真一致性验证。 |
| **PyBullet Visualizer**  | `telegrip/core/pybullet_visualizer.py`                       | 用于在 PyBullet 中实时可视化机械臂动作，实现 3D 动作预览与调试。 |
| **辅助功能模块**         | `telegrip/utils.py`、`kinematics.py`、`config.py`、`config.yaml` | 提供配置管理、IK解算算法、姿态文件解析、参数加载等工具功能。 |
| **入口脚本**             | `telegrip/main.py`                                           | 程序入口：启动 WebSocket 服务器、加载配置、创建控制循环、启动通信线程。 |

从代码角度看整体运行流程如下：

<img src="./assets/image-20251010161526693.png" alt="image-20251010161526693" style="zoom: 50%;" />

整体大致可分为采集、数据格式统一处理、解算并发送三个流程，其中采集采用 浏览器插件 A-Frame，其是一个**基于 HTML 的 WebXR 框架**，用于构建虚拟现实（VR）、增强现实（AR）和 3D 场景，运行在普通网页浏览器中。它是 Mozilla 开发的开源项目，目标是让 **任何人都能通过写 HTML 快速搭建 3D/VR 应用**，此处是使用其进行手柄位置与姿态的采集，同时在该部分使用 websocket 将在 VR 端采集到的数据发送服务端；

接着在服务端 vr_ws_server.py 的进行接收并进行数据的统一处理，注意此处仅包括 VR 端数据，键盘数据因为是本地监听，在监听时就已经做了处理；

处理完成后通过 python 内部通信传输到 control_loop.py ，在 pybullet 中进行 IK 解算，解算完成后将对应的位姿发送到 pybullet 的仿真以及外部实际的硬件中。

各部分详细处理大致如下所示：

![image-20251010163456841](./assets/image-20251010163456841.png)

此处对应数据采集部分；

![image-20251010163521580](./assets/image-20251010163521580.png)

此处对应数据统一处理并发送；

![image-20251010163714663](./assets/image-20251010163714663.png)

此处对应数据 IK 解算、数据发送；

# 三、小结与运行效果

TeleGrip 项目以 **浏览器端 VR 控制 → WebSocket 数据传输 → 后端控制循环解算 → 机械臂动作执行** 为核心流程，实现了从虚拟空间到物理空间的实时映射。其系统结构层次清晰、模块划分明确、易于扩展，是一个轻量化的 VR 远程操控框架。

通过 TeleGrip，可以在普通 PC + Meta Quest + 入门级 SO100 机械臂的低成本配置下，实现高精度、低延迟的远程动作控制。
 前端基于 A-Frame 框架进行位姿采集与可视化，后端以 Python 为主线，实现了命令队列管理、控制循环计算、运动插值与姿态解算，并支持 PyBullet 仿真与实际硬件同步。

该框架不仅可用于 **VR 操控实验、具身智能交互研究、双臂协作任务验证**，也可作为后续多模态交互系统（如语音控制、视觉识别、策略优化） 的基础模块，此外 LeRobot 本身也自带许多可进行研究的模块，包括 VLA 模型的训练、微调等。

下图为 TeleGrip 的运行效果示例，展示了 **VR 控制器驱动 SO100 机械臂进行同步运动** 的过程：

<img src="./assets/output.gif" alt="output" style="zoom:67%;" />

可以看出在大幅度动作时，有些动作对应不上，可能是采用的 IK 解算方法为比较基础的 DLS（阻尼最小二乘法）的关系，也可能是中间数据没有进行插值的原因（简单插值后动作会顺滑一些），不过此处仅作验证，不做过多概述。