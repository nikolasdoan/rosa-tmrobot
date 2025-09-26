## ROSA + TM Robot Integration (TM Workspace)

This document explains all changes and steps to integrate a ROSA-based agent to control a TM robot in the `tm2_ros2-humble` workspace.

### Prerequisites
- TM workspace: `/home/asrlab/tm2_ros2-humble` (fresh clone or equivalent)
- Model-specific TM MoveIt package (e.g., `tm5s_moveit_config`) present in the workspace
- ROS 2 Humble installed
- Python 3.10 with the following packages installed in your environment:
  ```bash
  pip install langchain rich pyinputplus python-dotenv
  # If 'rosa' is not installed via pip, install from its source as documented by ROSA
  # Example (adjust to your ROSA source):
  # pip install -e /path/to/rosa
  ```
  Notes:
  - The agent imports `rosa`, `langchain`, `rich`, `pyinputplus`, and `python-dotenv`.
  - If your LLM backend needs credentials (e.g., `OPENAI_API_KEY`), create a `.env` file in your home or workspace root with the key, or export it in the shell.

### High-level approach
- Use TM’s `tmr_arm_controller` (JointTrajectoryController) for joint commands
- Avoid custom controller switchers or external cartesian services
- Keep agent Python tools and run them from source (no full rebuild needed)

---

## 1) Build and source the TM workspace

If this is a fresh clone, build once and source:
```bash
cd /home/asrlab/tm2_ros2-humble
colcon build
source install/setup.bash
```

## 2) Launch the TM stack (MoveIt + controllers)

Pick your TM model and launch its MoveIt file. Example for TM5S:

```bash
source /home/asrlab/tm2_ros2-humble/install/setup.bash
ros2 launch tm5s_moveit_config tm5s_moveit.launch.py
```

This starts:
- `move_group`
- `rviz2` with TM config
- `robot_state_publisher`
- `controller_manager/ros2_control_node`
- `joint_state_broadcaster` (active)
- `tmr_arm_controller` (active)

Verify controllers:
```bash
ros2 control list_controllers | cat
# expect: tmr_arm_controller [active], joint_state_broadcaster [active]

ros2 topic list | grep tmr_arm_controller | cat
# expect: /tmr_arm_controller/joint_trajectory

ros2 action list | grep follow_joint_trajectory | cat
# expect: /tmr_arm_controller/follow_joint_trajectory
```

---

## 3) Agent code changes (TM-specific)

All edits are in the TM workspace copy of the agent under:
`/home/asrlab/tm2_ros2-humble/src/llm-ur-control/ur_agent`

### 3.1 Update the joint command publisher and joint names
File: `scripts/tools/ur.py`
- Set publisher topic to `/tmr_arm_controller/joint_trajectory`
- Use TM joint names and ordering: `joint_1` .. `joint_6`
- Use the joint state message order as-is:
  - `current_joint_states = list(msg.position)`

Resulting behavior:
- The agent publishes `trajectory_msgs/JointTrajectory` with `joint_1..joint_6` to `/tmr_arm_controller/joint_trajectory`.

### 3.2 Disable unused tools (switcher + cartesian)
File: `scripts/tools/ur.py`
- Stubbed to return a clear Not Supported message:
  - `activate_controller_request` → Not supported (use `controller_manager`; no custom switcher)
  - `cartesian_motion_request` → Not supported (no cartesian service wired here)
  - `get_current_pose` → Not supported (no cartesian pose topic wired here)

### 3.3 Blacklist unused tools at the agent layer
File: `scripts/ur_agent.py`
- In `URAgent.__init__`: set
  ```python
  self.__blacklist = [
      "activate_controller_request",
      "cartesian_motion_request",
      "get_current_pose",
  ]
  ```
- Update examples and greeting to TM context.

### 3.4 Update system prompts to TM
File: `scripts/prompts.py`
- Persona mentions TM (e.g., TM5S)
- Critical instructions:
  - Use `tmr_arm_controller` for joint motions
  - Joints listed as `joint_1..joint_6`
- Capabilities reflect joint-only control; no controller switching/cartesian services

---

## 4) Running the agent (TM)

Option A: Run from source (no rebuild needed for Python changes)
```bash
source /home/asrlab/tm2_ros2-humble/install/setup.bash
python3 /home/asrlab/tm2_ros2-humble/src/llm-ur-control/ur_agent/scripts/ur_agent.py
```

Option B: Using ros2 run (rebuild just the package)
```bash
cd /home/asrlab/tm2_ros2-humble
colcon build --packages-select ur_agent
source install/setup.bash
ros2 run ur_agent ur_agent.py
```

Notes:
- Do NOT launch `ur_agent/launch/agent.launch.py`; it starts services not used here (`controller_switcher.py`, `cartesian_motion_server.py`).
- Ensure the TM MoveIt launch is already running so that `/tmr_arm_controller` is active.

---

## 5) Example interaction

With the TM MoveIt stack running, start the agent and ask:
```
move joint 4 90 degree
```
Expected behavior:
- Agent converts degrees→radians
- Reads `/joint_states`
- Publishes `JointTrajectory` to `/tmr_arm_controller/joint_trajectory` with joint_1..joint_6
- RViz updates the robot pose accordingly

---

## 6) Troubleshooting

- No movement visible in RViz
  - Confirm `robot_state_publisher` is running (it is part of the TM launch)
  - Confirm joint names/ordering are `joint_1..joint_6` and not UR names
  - Echo controller feedback/state:
    ```bash
    ros2 topic echo /tmr_arm_controller/state | head -n 20 | cat
    ```

- Agent tries to switch controllers
  - Ensure you are running the TM agent version (greeting: “ROSA-TM agent”) and the blacklist is in effect
  - Source only the TM workspace (or source it last) before `ros2 run`

- LLM credentials or connectivity issues
  - If the agent fails to respond or stream LLM output, ensure your `.env` contains required keys (e.g., `OPENAI_API_KEY`) and your network allows access. Alternatively, configure `get_llm()` for a local/offline model.

 

---

## 7) Files changed (TM workspace)

- `src/llm-ur-control/ur_agent/scripts/tools/ur.py`
  - Publisher topic → `/tmr_arm_controller/joint_trajectory`
  - Joint names → `joint_1..joint_6`
  - Joint state callback uses message order
  - UR-only tools return Not Supported

- `src/llm-ur-control/ur_agent/scripts/ur_agent.py`
  - Blacklist UR-only tools
  - Examples + greeting updated to TM

- `src/llm-ur-control/ur_agent/scripts/prompts.py`
  - Persona updated to TM
  - Critical instructions: `tmr_arm_controller`, joints `joint_1..joint_6`
  - Capabilities: joint control only

---

## 8) What we intentionally did NOT use

- Custom controller switchers
- External cartesian controllers/services not wired in this repo
- `ur_agent/launch/agent.launch.py` (not required here)

---

## 9) Next steps (optional)

- If you want cartesian motion for TM through the agent, add a TM-compatible cartesian controller and expose:
  - A pose service (e.g., `move_to_pose`) and/or a pose topic
  - Then wire new tools in `scripts/tools/ur.py` and remove from blacklist
