# Mecanum Robot Digital Twin — Isaac Sim 5.1 + ROS2 Humble

Real-time digital twin of a mecanum wheel AMR. The physical robot streams live encoder and IMU data via micro-ROS over WiFi. A ROS2 Humble stack on Ubuntu processes the data and forwards it to NVIDIA Isaac Sim 5.1 on Windows via FastDDS — the simulation mirrors wheel velocities in real time.

---

## Demo

- ✅ Forward / Backward movement sync
- ✅ 360° rotation sync
- ✅ IMU + wheel odometry EKF fusion
- ✅ RViz visualization with TF tree
- ✅ Isaac Sim digital twin receiving live `/joint_states` via ROS2 bridge

---

## Hardware

| Component | Specification |
|---|---|
| Microcontroller | Arduino Portenta H7 |
| Motor Controllers | Dual RoboClaw 2x15A |
| Motors | goBILDA 5203 (19.2:1, 312 RPM) |
| Wheels | goBILDA Mecanum 96mm (2 LH + 2 RH) |
| Wheel Radius | 48 mm |
| Encoder CPR | 2150 counts/rev |
| IMU | SparkFun ISM330DHCX |
| Chassis | goBILDA Strafer Chassis Kit v4 (4.966 kg) |
| Communication | micro-ROS over WiFi UDP |

---

## Software Stack

### Ubuntu (10.42.0.1)
- Ubuntu 22.04 LTS
- ROS2 Humble Hawksbill
- micro-ROS Agent (Docker)
- robot_localization (EKF)
- RViz2

### Windows (10.42.0.71)
- Windows 11
- NVIDIA Isaac Sim 5.1.0
- GPU Driver 580.88 WHQL (RTX 5080)
- FastDDS cross-machine bridge

---

## Network

```
PortentaROS WiFi Hotspot
├── Ubuntu laptop   — 10.42.0.1  (ROS2 master)
├── Windows desktop — 10.42.0.71 (Isaac Sim)
└── Portenta H7     — 10.42.0.128 (micro-ROS agent port 8888)
```

---

## ROS2 Topics

| Topic | Type | Description |
|---|---|---|
| `/cmd_vel` | geometry_msgs/Twist | Velocity commands |
| `/joint_states` | sensor_msgs/JointState | Wheel encoder data from robot |
| `/joint_states_fixed` | sensor_msgs/JointState | Mirrored + scaled for Isaac Sim |
| `/odom` | nav_msgs/Odometry | Raw wheel odometry |
| `/odometry/filtered` | nav_msgs/Odometry | EKF fused odometry |
| `/imu` | sensor_msgs/Imu | IMU data |
| `/tf` | tf2_msgs/TFMessage | Transform tree |

---

## Files

| File | Description |
|---|---|
| `joint_state_fix.py` | Mirrors FR/RR wheel signs + velocity scale for Isaac Sim |
| `digital_twin.py` | Publishes TF and joint states for RViz |
| `teleop_qweasdzxc.py` | Keyboard teleop (QWEASDZXC + Space) |
| `odom_bridge.py` | Optional: publishes pose to Isaac Sim via /isaac_robot_pose |
| `fastdds_config.xml` | FastDDS peer config for Windows → Ubuntu DDS discovery |

---

## Startup Sequence

### Ubuntu — 8 terminals

```bash
# T1 — Hotspot
nmcli connection up Hotspot

# T2 — micro-ROS agent
docker run -it --rm --net=host microros/micro-ros-agent:humble udp4 --port 8888 -v4

# T3 — robot_state_publisher
source /opt/ros/humble/setup.bash
ros2 run robot_state_publisher robot_state_publisher --ros-args \
  -p robot_description:="$(cat ~/mecanum_viz_ws/src/mecanum_description/urdf/mecanum_robot.urdf)"

# T4 — EKF
source /opt/ros/humble/setup.bash
ros2 run robot_localization ekf_node --ros-args \
  --params-file ~/mecanum_viz_ws/src/mecanum_description/config/ekf.yaml

# T5 — digital_twin
source /opt/ros/humble/setup.bash
python3 ~/digital_twin.py

# T6 — joint_state_fix
source /opt/ros/humble/setup.bash
python3 ~/joint_state_fix.py

# T7 — RViz
source /opt/ros/humble/setup.bash
ros2 run rviz2 rviz2

# T8 — Teleop
source /opt/ros/humble/setup.bash
python3 ~/Desktop/teleop_qweasdzxc.py
```

### Windows — Isaac Sim

```cmd
C:\isaacsim\isaac-sim-standalone-5.1.0-windows-x86_64\isaac-sim.bat --/isaac/startup/ros_bridge_extension=isaacsim.ros2.bridge
```

### Windows — FastDDS environment variable (run once as Admin)

```powershell
[System.Environment]::SetEnvironmentVariable("FASTRTPS_DEFAULT_PROFILES_FILE", "C:\Users\ricar\fastdds_config.xml", "Machine")
```

---

## Isaac Sim Setup

### USD Scene Structure
```
/World/robot_m
└── base_link          ← Rigid Body, Mass 4.966 kg, Articulation Root
└── wheel_front_left   ← Rigid Body, Mass 0.207 kg
    └── fl_wheel_joint ← RevoluteJoint, Axis=Y, Angular Drive
└── wheel_front_right  ← fr_wheel_joint
└── wheel_rear_left    ← rl_wheel_joint
└── wheel_rear_right   ← rr_wheel_joint
```

### Angular Drive Parameters
```
Type:       force
Stiffness:  0.0
Damping:    10000.0
Max Force:  100000.0
```

### Action Graph
- On Playback Tick → ROS2 Subscribe Joint State (`/joint_states_fixed`) → Articulation Controller (`/World/robot_m`)

---

## Known Limitations

- Mecanum roller physics uses full mesh colliders — individual roller colliders per body needed for accurate lateral motion simulation
- Lateral odometry drift accumulates due to mecanum wheel kinematics
- Velocity SCALE factor in `joint_state_fix.py` needs tuning per movement type

---

## Next Steps

- Individual roller colliders with proper joint placement per roller
- Isaac Lab RL training pipeline using this digital twin
- Sim-to-real transfer for autonomous navigation

---

## Author

Ricardo Manriquez  
Senior Data Analyst — Peterson CAT  
UC Berkeley MAS-E — AI/ML, Robotics & Controls (Expected Dec 2026)  
[LinkedIn](https://www.linkedin.com/in/ricardomanriquez)
