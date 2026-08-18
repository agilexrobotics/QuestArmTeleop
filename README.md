<div align="center">
  <h1 align="center"> quest teleop piper </h1>
  <h3 align="center"> Agilex Robotics </h3>
  <p align="center">
    <a>English</a> | <a href="README_zh_CN.md">中文</a> 
  </p>
</div>


## Introduction

This repository implements teleoperation of Agilex robotic arms using Meta Quest 2/3/3S VR headsets.

### Prerequisites

**1. Install dependencies and clone the code**

Install dependencies:

```bash
sudo apt install android-tools-adb

conda create -n vt python=3.10.12

conda activate vt

conda install pinocchio==3.2.0  -c conda-forge

pip install meshcat casadi pyyaml pure-python-adb
```

Clone the code and build:

```bash
git clone   https://github.com/agilexrobotics/QuestArmTeleop.git

cd QuestArmTeleop/src

git clone https://github.com/agilexrobotics/agx_arm_ros.git

cd agx_arm_ros/src/agx_arm_description

git clone -b flattened https://github.com/agilexrobotics/agx_arm_urdf.git

cd ~/QuestArmTeleop

colcon build
```

**2. Enable Developer Mode (required; otherwise third-party APKs cannot be installed)**

Before you begin, check whether Developer Mode is already enabled on your Quest device:

Settings → Advanced → Developer → turn on **Enable Developer Settings**.

If it is already enabled, skip this step.

If you do not see Developer options, follow the steps below to activate them.

1. Register a Meta developer account
   → Visit the Meta Developer Platform, sign in with your Meta account, create an organization (any name is fine), and complete verification by linking a credit card.

2. Enable Developer Mode in the mobile app
   → Open the Meta Quest app on your phone → Device Settings → Developer Mode → turn on the switch.

3. Allow unknown sources on the headset
   → On the headset, go to Settings → System → Developer Options → enable **Unknown Sources**.

**3. Set headset sleep timeout**

Set the sleep timeout to the maximum value so the headset does not turn off the display and stop publishing pose data.

→ On the headset, go to Settings → General → Power → set **Display Off Time** to 4 hours.

**4. Turn off gesture tracking and recognition to ensure that only the controllers are detected.**
**5. Connect the computer and Quest with a USB Type-C cable**

Wired connection is the default, because it provides reliable data throughput and low latency. If you need wireless connection, see [Wireless Connection](#wireless-connection).

**6. Install the APK on the headset**

- Establish the connection: after enabling Developer Mode, connect the Quest to your computer with a USB cable → when the **Allow USB debugging** prompt appears on the Quest → authorize to establish the channel.
- Run the following command:

```bash
adb install ~/QuestArmTeleop/src/oculus_reader/APK/teleop-debug.apk
```

Wait until the terminal prints `Success`, which means installation succeeded.

### Code Architecture

`oculus_reader` provides tools for reading poses and button presses from the Quest device.

Runtime flow: `pub_pose.py` first publishes controller pose data; `pub_delta_pose.py` subscribes to that data, processes it, and publishes delta pose while using the trigger buttons to control the gripper; finally, `arm_ik_pose_node.py` subscribes to delta pose, solves IK for the main arm joints, and publishes them as topics for the robot arm to follow.

```bash
.
├── img
│   ├── 1.png
│   └── 2.png
├── README.md
└── src
    ├── agx_arm_ros
    │   ├── scripts   # CAN module activation scripts
    │   │   ├── agx_arm_install_deps.sh
    │   │   ├── can_activate.sh        
    │   │   ├── can_config.sh
    │   │   ├── can_muti_activate.sh
    │   │   └── find_all_can_port.sh
    │   └── src
    │       ├── agx_arm_ctrl  # Robot arm control
    │       └── agx_arm_description  # Robot arm model files
    └── oculus_reader
        ├── APK  # Headset software
        │   ├── alvr_client_android.apk
        │   └── teleop-debug.apk
        ├── CMakeLists.txt
        ├── config
        │   ├── arm_ik_pose_node.nero.yaml  # Nero IK config
        │   ├── arm_ik_pose_node.piper_x.yaml  # Piper X config
        │   └── oculus_reader.rviz
        ├── launch
        │   ├── teleop_single_nero.launch.py   # Launch Nero teleop
        │   └── teleop_single_piper_x.launch.py  # Launch Piper X teleop
        ├── package.xml
        └── scripts
            ├── arm_ik_pose_node.py  # IK core file
            ├── buttons_parser.py  # Controller button handling
            ├── FPS_counter.py
            ├── install.py
            ├── oculus_reader.py  
            ├── pub_delta_pose.py  # Process controller pose and publish delta pose
            ├── pub_pose.py  # Publish controller pose topic
            └── transformations.py
```

## Software Startup

1. Activate the CAN module

**Activate a single CAN module**

When only one CAN module is connected to the computer, activate it with:

```bash
bash ~/QuestArmTeleop/src/agx_arm_ros/scripts/can_activate.sh 
```

**Activate dual CAN modules**

First connect the CAN module for the left arm to the computer, then run:

```bash
bash ~/QuestArmTeleop/src/agx_arm_ros/scripts/find_all_can_port.sh 
```

The terminal will show the CAN port for the left arm. Then connect the CAN module for the right arm.

Run again:

```bash
bash ~/QuestArmTeleop/src/agx_arm_ros/scripts/find_all_can_port.sh 
```

The terminal will show the CAN port for the right arm.

Copy the left and right port names into lines 111 and 112 of `can_config.sh`, as shown below:

```python
if [ "$EXPECTED_CAN_COUNT" -ne 1 ]; then
    declare -A USB_PORTS 
    USB_PORTS["1-8.1:1.0"]="can_left:1000000"  # Left CAN
    USB_PORTS["1-8.2:1.0"]="can_right:1000000" # Right CAN
fi
```

After saving, activate both arms:

```bash
bash ~/QuestArmTeleop/src/agx_arm_ros/scripts/can_config.sh 
```

2. Start teleoperation

Before starting teleoperation, read the [Operation Guide](#operation-guide).

```bash
source ~/QuestArmTeleop/install/setup.bash 

conda activate vt

# Start single-arm Nero teleop (after launching, do not start teleoperation immediately; read the sections below first)
ros2 launch  oculus_reader teleop_single_nero.launch.py 

# Start dual-arm Nero teleop (after launching, do not start teleoperation immediately; read the sections below first)
ros2 launch  oculus_reader teleop_single_nero.launch.py 

# Start Piper X teleop (after launching, do not start teleoperation immediately; read the sections below first)
ros2 launch  oculus_reader teleop_single_piper_x.launch.py
```

After launching single-arm teleop, two RViz windows appear: one shows the controller and VR headset coordinates, and the other shows the robot arm model. The model subscribes to live joint feedback from the real arm so the simulated and physical joint states stay synchronized.

Wearing setup: hang the VR headset around your neck and hold the left and right controllers with the joysticks facing upward.

In the RViz window that shows the controller and VR headset frames, the three frames should look like this after you wear the equipment as described:

![img error](img/3.png)

**If the left or right controller frame is not in the expected quadrant, keep moving the controller until it settles into the correct position.**

Teleoperation maps the controller pose to the robot gripper end-effector, so the frames must be aligned. Otherwise, moving the controller left may cause the arm to move right.

In the RViz window that shows the robot arm, with the initial pose of joint2 at -30° and joint4 at 120°, the end-effector frame should match the controller frame:

![img error](img/4.png)

At this point, you can start teleoperation.

If the arm end-effector frame is not aligned with the controller frame, adjust the `ros_to_arm_rpy` parameter in the `pub_pose_node` node inside the launch file.

If you see this error when starting teleoperation:

```bash
Device not found. Make sure that device is running and is connected over USB
Run `adb devices` to verify that the device is visible.
```

the VR headset is not in debugging mode. Enable it as follows:

1. Connect the VR headset to the computer with a USB-C cable and put on the headset.

2. When the **USB detected** notification appears, tap it.

   ![img error](img/2.png)

3. The first time you start the program, you may see the error above.

4. When prompted on the device, accept **Allow USB debugging** or tap **Always allow from this computer**.

   ![img error](img/1.png)

5. Close the program and run it again.



## Operation Guide

> ⚠️ Notes:
> - For Nero arms, wait until the power indicator turns green before starting teleoperation. Piper-series arms do not require this wait.
> - Keep the VR display awake. If the screen turns off, pose data will drift and the arm may move unexpectedly. We recommend covering the proximity sensor inside the headset to keep the display on.
> - After launching the program, make sure the controllers stay in the VR field of view and that the frames in RViz are stable. The controller TF frames must also be aligned with the arm end-effector frames (adjust `ros_to_arm_rpy` in the launch file) before starting teleoperation.
> - Aligning the controller and arm end-effector frames is critical for a good teleoperation experience.
> - For single-arm control, use the right controller: hold **A** to start teleoperation and **B** to stop.
> - For dual-arm control: on the right controller, hold **A** to start / **B** to stop the right arm; on the left controller, hold **X** to start / **Y** to stop the left arm.
> - This program supports wireless teleoperation. Performance depends on your network connection quality.

## Controller Button Reference

Button values can be obtained with:

```bash
transformations, buttons = oculus_reader.get_transformations_and_buttons()
```

The following is one frame of button data printed with `print("buttons:", buttons)`:

```python
buttons: {'A': False, 'B': False, 'RThU': True, 'RJ': False, 'RG': False, 'RTr': False, 'X': False, 'Y': False, 'LThU': True, 'LJ': False, 'LG': False, 'LTr': False, 'leftJS': (0.0, 0.0), 'leftTrig': (0.0,), 'leftGrip': (0.0,), 'rightJS': (0.0, 0.0), 'rightTrig': (0.0,), 'rightGrip': (0.0,)}
```

### Button States (Booleans: `True`/`False`)

These values are booleans (`True` or `False`). `False` means the button is not pressed; `True` means it is pressed.

- **`'A': False`**: Right controller **A** button is not pressed.
- **`'B': False`**: Right controller **B** button is not pressed.
- **`'X': False`**: Left controller **X** button is not pressed.
- **`'Y': False`**: Left controller **Y** button is not pressed.
- **`'RThU': True`**: **R**ight **Th**umbstick **U**p. Your right thumb is resting on the capacitive sensor of the right joystick, but the stick is not clicked.
- **`'LThU': True`**: **L**eft **Th**umbstick **U**p. Your left thumb is resting on the capacitive sensor of the left joystick, but the stick is not clicked.
- **`'RJ': False`**: **R**ight **J**oystick (or thumbstick) click. The right joystick is not pressed.
- **`'LJ': False`**: **L**eft **J**oystick (or thumbstick) click. The left joystick is not pressed.
- **`'RG': False`**: **R**ight **G**rip. The right grip button (pressed by the middle finger) is not pressed.
- **`'LG': False`**: **L**eft **G**rip. The left grip button (pressed by the middle finger) is not pressed.
- **`'RTr': False`**: **R**ight **Tr**igger. The right trigger (pressed by the index finger) is not fully pressed (a threshold is usually used to determine `True`).
- **`'LTr': False`**: **L**eft **Tr**igger. The left trigger (pressed by the index finger) is not fully pressed.

### Joystick and Analog Sensor Values (Tuples with Floats)

These values are float tuples representing joystick deflection or trigger/grip press depth, usually in the range 0.0 to 1.0, or -1.0 to 1.0 for joysticks.

- **`'leftJS': (0.0, 0.0)`**: Left joystick state. This is a tuple `(x, y)` for horizontal and vertical deflection. `(0.0, 0.0)` means the joystick is centered.
- **`'rightJS': (0.0, 0.0)`**: Right joystick state. Same as above; `(0.0, 0.0)` means centered.
- **`'leftTrig': (0.0,)`**: Left trigger press depth. `0.0` means fully released; `1.0` means fully pressed.
- **`'rightTrig': (0.0,)`**: Right trigger press depth. `0.0` means fully released.
- **`'leftGrip': (0.0,)`**: Left grip press depth. `0.0` means fully released; `1.0` means fully pressed.
- **`'rightGrip': (0.0,)`**: Right grip press depth. `0.0` means fully released.



## Wireless Connection

### Phase 1: Preparation (must be on the same LAN)

- **Same Wi-Fi**: Make sure your computer and Quest are connected to the same router.
- **Prefer 5 GHz**: To reduce latency and packet loss, strongly prefer **5 GHz** Wi-Fi over 2.4 GHz.
- **ADB on the computer**: Make sure `adb` is installed on your computer.

### Phase 2: First Connection and Enable Wireless Mode

Quest disables the wireless debugging port by default after a reboot, so you usually need to run the steps below **after every full power cycle**:

1. **Connect with USB**: Connect the Quest to your computer with a USB cable.

2. **Authorize the device**: Put on the headset. If prompted with **Allow USB debugging?**, check **Always allow** and confirm.

3. **Enable TCP mode**: On your computer, run:

   ```bash
   adb tcpip 5555
   ```

   *If successful, the terminal returns: `restarting in TCP mode port: 5555`.*

4. **Disconnect USB**: You can now unplug the cable.

### Phase 3: Get the IP Address and Connect Wirelessly

1. **Find the Quest IP address**:

   - **Method A (on the headset)**: Settings -> Wi-Fi -> tap the connected network -> Details -> note the headset IP address.

   - **Method B (command line)**:

     ```bash
     adb shell ip route
     ```

     *Look for the number after `src` on the `wlan0` line.*

2. **Connect wirelessly** (this helps ensure the Python script runs correctly):

   ```bash
   adb connect <YOUR_QUEST_IP>:5555
   # Example: adb connect 192.168.1.101:5555
   ```

   *If you see `connected to ...`, the wireless link is established.*

### Phase 4: Use in Python

In your code, pass the same IP address:

```Python
from oculus_reader import OculusReader

# Make sure this IP matches the one used in adb connect
self.oculus_reader = OculusReader(ip_address='192.168.1.101') 
```

### Phase 5: Run the Program

`oculus_reader` depends on an APK installed on the Quest to capture sensor data.

1. **Make sure the APK is installed**
2. **Start the application**:
   - Follow [Software Startup](#software-startup) to launch the program.
   - After launch, you may be prompted with **Allow USB debugging?** Check **Always allow** and confirm.
