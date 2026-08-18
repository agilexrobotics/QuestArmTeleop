<div align="center">
  <h1 align="center"> quest teleop piper </h1>
  <h3 align="center"> Agilex Robotics </h3>
  <p align="center">
    <a href="README.md"> English </a> | <a href="README_zh_CN.md">中文</a> 
  </p>
</div>


## 介绍

该仓库实现了使用 meta quest2/3/3S VR 套装对本公司各款机械臂进行遥操作。

### 准备工作 

**一、安装依赖并克隆代码**

安装依赖：
```bash
sudo apt install android-tools-adb

conda create -n vt python=3.10.12

conda activate vt

conda install pinocchio==3.2.0  -c conda-forge

pip install meshcat casadi pyyaml pure-python-adb
```

将代码克隆下来并编译：

```bash
git clone   https://github.com/agilexrobotics/QuestArmTeleop.git

cd QuestArmTeleop/src

git clone https://github.com/agilexrobotics/agx_arm_ros.git

cd agx_arm_ros/src/agx_arm_description

git clone -b flattened https://github.com/agilexrobotics/agx_arm_urdf.git

cd ~/QuestArmTeleop

colcon build
```

**二、开启开发者模式（必须步骤，否则无法安装第三方APK）**

开始前请确认自己 quest 设备中是否有开发者模式，请参考如下步骤查找：

设置 → 高级 → 开发者 → 将 "启用开发者设置" 打开。

如果有，则跳过此步。

如果没有开发者选项，则参考下面步骤进行激活。

1、注册Meta开发者账号
→ 访问 Meta开发者平台，用Meta账号登录后创建组织（名称随意），绑定信用卡完成验证。

2、在手机App中开启开发者模式
→ 打开手机端 Meta Quest App → 设备设置 → 开发者模式 → 开启开关。

3、在头显中允许未知来源
→ 头显内进入 设置 → 系统 → 开发者选项 → 开启 "未知来源"权限。

**三、设置头显休眠时长**

需要将休眠时长设置最大，以免头显息屏导致无法输出位姿数据。

→ 头显内进入 设置 → 常规 → 电源 → 将"显示屏关闭时间" 调成 4 小时。

**四、 将手势跟踪识别关掉，确保其只识别手柄**

**五、使用USB-typeC线将电脑与quest设备连接**

默认是使用有线连接，因为有线连接能保证数据传输的速率以及做到低延迟，如果有无线连接的需求，点击跳转至[无线连接](#无线连接)查看。

**六、在头显中安装 APK 文件**
- 建立连接：开启开发者模式后，用数据线连接Quest与电脑 → Quest弹出"允许USB调试"提示 → 授权后建立通道
- 命令行输入：

```bash
adb install ~/QuestArmTeleop/src/oculus_reader/APK/teleop-debug.apk
```

等待一段时间后，终端输出 Success 即安装成功。

### 代码架构说明

oculus_reader，该存储库提供了从 Quest 设备读取位置和按下按钮的工具。

运行流程：首先从 pub_pose.py 获取到手柄 pose 数据，在 pub_delta_pose.py 订阅该数据并进行处理，发布 delta pose 数据同时利用手柄扳机键来控制夹爪开合，最后在 arm_ik_pose_node.py 中订阅 delta pose，IK解算成机械臂主体的几个关节角数据并以话题形式发出，机械臂订阅该话题完成控制。

```bash
.
├── img
│   ├── 1.png
│   └── 2.png
├── README.md
└── src
    ├── agx_arm_ros
    │   ├── scripts   # 激活can模块脚本
    │   │   ├── agx_arm_install_deps.sh
    │   │   ├── can_activate.sh        
    │   │   ├── can_config.sh
    │   │   ├── can_muti_activate.sh
    │   │   └── find_all_can_port.sh
    │   └── src
    │       ├── agx_arm_ctrl  # 机械臂控制
    │       └── agx_arm_description  # 机械臂模型文件
    └── oculus_reader
        ├── APK  # 头显软件
        │   ├── alvr_client_android.apk
        │   └── teleop-debug.apk
        ├── CMakeLists.txt
        ├── config
        │   ├── arm_ik_pose_node.nero.yaml  # nero IK 配置文件
        │   ├── arm_ik_pose_node.piper_x.yaml  # piper x 配置文件
        │   └── oculus_reader.rviz
        ├── launch
        │   ├── teleop_single_nero.launch.py   # 开启遥操nero程序
        │   └── teleop_single_piper_x.launch.py  # 开启遥操piper x程序
        ├── package.xml
        └── scripts
            ├── arm_ik_pose_node.py  # IK 核心文件 
            ├── buttons_parser.py  # 手柄按键处理
            ├── FPS_counter.py
            ├── install.py
            ├── oculus_reader.py  
            ├── pub_delta_pose.py  # 处理手柄pose数据并发出delta pose话题 
            ├── pub_pose.py  # 发布手柄 pose 话题
            └── transformations.py
```

## 软件启动

1、激活 CAN 模块

**激活单 CAN**：

当电脑仅连接单个 CAN 模块时，可通过以下步骤快速完成激活：

打开一个终端窗口，执行以下命令：

```bash
bash ~/QuestArmTeleop/src/agx_arm_ros/scripts/can_activate.sh 
```

**激活双 CAN 模块**：

先将连接左机械臂的的 CAN 模块接入电脑，

然后执行：

```bash
bash ~/QuestArmTeleop/src/agx_arm_ros/scripts/find_all_can_port.sh 
```

终端会出现左机械臂 CAN 的端口号，接着将右机械臂 CAN 模块接入电脑。

再次执行：

```bash
bash ~/QuestArmTeleop/src/agx_arm_ros/scripts/find_all_can_port.sh 
```

终端会出现右机械臂 CAN 的端口号。

将这左右两个端口号复制到 can_config.sh 文件的 111 和 112 行，如下所示：

```python
if [ "$EXPECTED_CAN_COUNT" -ne 1 ]; then
    declare -A USB_PORTS 
    USB_PORTS["1-8.1:1.0"]="can_left:1000000"  #左 CAN
    USB_PORTS["1-8.2:1.0"]="can_right:1000000" #右 CAN
fi
```

保存完毕后，激活左右机械臂使能脚本：

```bash
bash ~/QuestArmTeleop/src/agx_arm_ros/scripts/can_config.sh 
```

2、启动遥操机械臂

请在开始遥操前查看[操作说明](#操作说明)

```bash
source ~/QuestArmTeleop/install/setup.bash 

conda activate vt

# 启动遥操单nero（在开启遥操程序后，不要着急启动遥操，请仔细阅读下面的内容）
ros2 launch  oculus_reader teleop_single_nero.launch.py 

# 启动遥操双nero （在开启遥操程序后，不要着急启动遥操，请仔细阅读下面的内容）
ros2 launch  oculus_reader teleop_single_nero.launch.py 

# 启动遥操 piper x （在开启遥操程序后，不要着急启动遥操，请仔细阅读下面的内容）
ros2 launch  oculus_reader teleop_single_piper_x.launch.py
```

在启动单臂遥操后，会出现 2 个 RVIZ 的可视化界面，一个界面用来显示手柄与 VR 头显的坐标，一个用来显示机械臂的模型，该模型会实时订阅机械臂当前反馈的各关节数据来显示到模型上面，保证了真机和模型关节状态的同步。

穿戴方式：VR 头显佩戴在脖子上，双手握住左右手柄，摇杆面朝上。

先看显示手柄与 VR 头显坐标系的 RVIZ 界面，按上述要求佩戴后，3个坐标系应做到如下图所示：

![img error](img/3.png)

**如左右手柄坐标系不在相应的坐标系，则需不断晃动手柄直到位置收敛至指定象限即可。**

遥操是将手柄的 pose 值映射到机械臂夹爪末端上，所以要对其两者的坐标系，不然开启遥操后出现手柄往左边移动，机械臂往右边走的情况。

再来看显示机械臂的 RVIZ 界面，可以看到，在 joint2 为 -30°，joint4 为 120°的初始位姿下
的末端坐标系是跟手柄的坐标系是一样的：

![img error](img/4.png)

那么此时就可以开启遥操了。

如果机械臂的末端坐标系跟手柄不对齐，那么可通过修改 launch 文件中 pub_pose_node 节点中的 ros_to_arm_rpy 参数调整手柄的坐标系与机械臂末端坐标系对齐。

在启动遥操代码时出现该错误时：

```bash
Device not found. Make sure that device is running and is connected over USB
Run `adb devices` to verify that the device is visible.
```

说明了VR头盔未开启调试模式，开启调试模式方法步骤如下：

1. 使用 USB-C 线将VR头盔连接到计算机，然后佩戴该设备。

2. 当在通知中出现“检测到USB”，点击一下该通知。

   ![img error](img/2.png)

3. 第一次开启程序，会出现上面的报错。

4. 当设备上出现提示时，接受**允许 USB 调试**或点击**始终对这台电脑允许。**

   ![img error](img/1.png)

5. 关掉程序，再次运行。



## 操作说明

> 注意⚠️：
> - nero 机械臂需等电源处的指示灯变成绿色才可开启遥操程序，piper 系列机械臂无需等待。
> - 请一定要确保VR屏幕保持常亮，否则 pose 会乱飘导致遥操作机械臂乱飞，我们建议在VR眼镜里面拿东西遮住感应器，使其保持常亮状态。
> - 开启程序后，请一定要确保手柄在VR视野里以及rviz里面的坐标稳定不会乱飘，且手柄的TF坐标系是跟机械臂末端坐标系是对齐的（这部分需要在launch文件中修改ros_to_arm_rpy参数）方可开启遥操作。
> - 手柄与机械臂末端的坐标系对齐很重要，这直接决定了遥操的体验感。
> - 控制单臂需要用右手手柄，按住“A”键开始遥操，按住“B”键停止遥操。
> - 控制双臂时：右手柄按住“A”开始/“B”停止右臂；左手柄按住“X”开始/“Y”停止左臂。
> - 本程序支持无线遥操作，流程程度取决于你当下的网络连接速度。

## 手柄按键说明

按键值 button 可以通过下行代码获取到

```bash
transformations, buttons = oculus_reader.get_transformations_and_buttons()
```

以下数据是使用 `print("buttons:", buttons)` 打印 button 值的一帧数据：

```python
buttons: {'A': False, 'B': False, 'RThU': True, 'RJ': False, 'RG': False, 'RTr': False, 'X': False, 'Y': False, 'LThU': True, 'LJ': False, 'LG': False, 'LTr': False, 'leftJS': (0.0, 0.0), 'leftTrig': (0.0,), 'leftGrip': (0.0,), 'rightJS': (0.0, 0.0), 'rightTrig': (0.0,), 'rightGrip': (0.0,)}
```

### 按钮状态 (Booleans: `True`/`False`)

这部分的值是布尔类型（`True` 或 `False`），`False` 表示按钮未被按下，`True` 表示按钮被按下。

- **`'A': False`**: 右手柄的 "A" 按钮未被按下。
- **`'B': False`**: 右手柄的 "B" 按钮未被按下。
- **`'X': False`**: 左手柄的 "X" 按钮未被按下。
- **`'Y': False`**: 左手柄的 "Y" 按钮未被按下。
- **`'RThU': True`**: **R**ight **Th**umbstick **U**p。表示你的右拇指正放在右摇杆的电容传感器上，但并没有按下摇杆。
- **`'LThU': True`**: **L**eft **Th**umbstick **U**p。表示你的左拇指正放在左摇杆的电容传感器上，但并没有按下摇杆。
- **`'RJ': False`**: **R**ight **J**oystick (or Thumbstick) Click。右摇杆（拇指摇杆）没有被按下。
- **`'LJ': False`**: **L**eft **J**oystick (or Thumbstick) Click。左摇杆（拇指摇杆）没有被按下。
- **`'RG': False`**: **R**ight **G**rip。右侧握把键（中指按的键）没有被按下。
- **`'LG': False`**: **L**eft **G**rip。左侧握把键（中指按的键）没有被按下。
- **`'RTr': False`**: **R**ight **Tr**igger。右侧扳机键（食指按的键）没有被完全按下（通常有一个阈值来判断是否为 `True`）。
- **`'LTr': False`**: **L**eft **Tr**igger。左侧扳机键（食指按的键）没有被完全按下。

### 摇杆和传感器模拟值 (Tuples with Floats)

这部分的值是浮点数元组，表示摇杆的偏离程度或扳机/握把的按压深度，范围通常在 0.0 到 1.0 之间，或 -1.0 到 1.0 之间。

- **`'leftJS': (0.0, 0.0)`**: 左摇杆 (Left Joystick) 的状态。这是一个包含两个浮点数的元组 `(x, y)`，分别代表水平和垂直方向的偏离。`(0.0, 0.0)` 表示摇杆处于中心位置，没有被推动。
- **`'rightJS': (0.0, 0.0)`**: 右摇杆 (Right Joystick) 的状态。同上，`(0.0, 0.0)` 表示摇杆处于中心位置。
- **`'leftTrig': (0.0,)`**: 左扳机键 (Left Trigger) 的按压深度。`0.0` 表示完全松开，`1.0` 表示完全按下。
- **`'rightTrig': (0.0,)`**: 右扳机键 (Right Trigger) 的按压深度。`0.0` 表示完全松开。
- **`'leftGrip': (0.0,)`**: 左握把键 (Left Grip) 的按压深度。`0.0` 表示完全松开，`1.0` 表示完全按下。
- **`'rightGrip': (0.0,)`**: 右握把键 (Right Grip) 的按压深度。`0.0` 表示完全松开。



## 无线连接

### 第一阶段：准备工作（必须处于同一局域网）

- **同一 Wi-Fi**：确保你的电脑和 Quest 连接在同一个路由器的 Wi-Fi 下。
- **5GHz 优先**：为了降低延迟和数据丢包，强烈建议连接 **5G 频段** 的 Wi-Fi，而不是 2.4G。
- **电脑端工具**：确保电脑已安装 `adb` 工具。

### 第二阶段：首次连接与激活无线模式

Quest 在重启后默认会关闭无线调试端口，因此**每次 Quest 彻底关机重启后**，你通常需要执行一次以下步骤：

1. **USB 线连接**：用 USB 线将 Quest 连接到电脑。

2. **授权设备**：戴上头显，如果弹出“允许 USB 调试吗？”，勾选“始终允许”并确认。

3. **开启监听端口**：在电脑终端输入以下命令：

   ```bash
   adb tcpip 5555
   ```

   *如果成功，终端会返回：`restarting in TCP mode port: 5555`。*

4. **拔掉 USB 线**：现在你可以断开物理连线了。

### 第三阶段：获取 IP 并建立无线握手

1. **查询 Quest IP 地址**：

   - **方法 A (头显内)**：设置 -> Wi-Fi -> 点击已连接的 Wi-Fi -> 详情 -> 记录下头显的 IP 地址。

   - **方法 B (电脑命令行)**：

     ```bash
     adb shell ip route
     ```

     *查看 `wlan0` 对应的 `src` 后面的数字。*

2. **手动建立无线连接**（这一步能确保 Python 脚本顺利运行）：

   ```bash
   adb connect <你的Quest_IP>:5555
   # 示例：adb connect 192.168.1.101:5555
   ```

   *看到 `connected to ...` 说明无线链路已打通。*

### 第四阶段：Python 代码调用

在你的代码中，直接填入该 IP 即可：

```Python
from oculus_reader import OculusReader

# 确保这里的 IP 与上面 adb connect 的 IP 完全一致
self.oculus_reader = OculusReader(ip_address='192.168.1.101') 
```

### 第五阶段：运行程序

`oculus_reader` 依赖于安装在 Quest 里的一个 APK 文件来抓取传感器数据。

1. **确保已安装 APK**
2. **启动应用程序：
   - 参考[软件启动](#软件启动)开启动程序。
   - 程序启动后可能会弹出“允许 USB 调试吗？”，勾选“始终允许”并确认。
