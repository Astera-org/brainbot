# brainbot

Brainbot is a modular hub-and-spoke control stack for combining teleoperation, AI inference, multi-camera streaming, and lightweight visualization. A typical deployment uses a **remote_host** (e.g., a Jetson module mounted on the robot) to run the follower robot and AI policy, while a **hub_host** launches leader teleop devices or AR controllers.

This pipeline also includes the following repos:
- Manipulation RL in IsaacLab https://github.com/brain-sim/manipulation-lab
- AR App https://github.com/SMSL-Project/unity-so101
- Customized LeRobot Fork https://github.com/bingogome/lerobot
- Customized GR00T Fork https://github.com/bingogome/Isaac-GR00T
- IsaacSim Asset Package (coming soon)
- Datasets (coming soon)

## Architecture

```
Command Providers -> Hub:                   |    Command Consumers:
┌───────────────┐     ┌──────────────────┐  |            ┌───────────────────────┐
│ Teleop Server │◄───►│ Command Service  │  |            │ Web Dashboard         │
│(on hub_host)  │ ZMQ │ (remote_host)    │  |            │ (brainbot_webviz)     │
│ run_teleop_…  │     └──────────────────┘  |            └───────────────────────┘
└───────────────┘                           |                      ▲
                                            |                      │ HTTP
┌───────────────────────────────────┐       |                      │
|  AI Policy Server (remote_host)   │       |      ┌───────────────┴─────────────────┐
└───────────────────────────────────┘       |      │ Command Service (remote_host)   │
                                            |      │ (brainbot_command_service)      │
                                            |      │ Mode manager + CLI              │
                                            |      │  • Local/remote teleop provider │
                                            |      │  • AI provider                  │
                                            |      │  • WebViz hook                  │
                                            |      └────────────▲──────────────┬─────┘
                                            |                   │ ZMQ          │
                                            |                   │              │
                                            |      ┌────────────┴──────────────▼──────┐
                                            |      │ Robot Controller (remote_host)   │
                                            |      │ (brainbot_control_service)       │
                                            |      │  • Streams obs/actions           │
                                            |      │  • Drives hardware               |
                                            |      │  • Camera tap                    |
                                            |      └──────────────────────────────────┘                                          
```

## Packages

| Package                    | Description                                                   |
|----------------------------|---------------------------------------------------------------|
| `brainbot_core`            | Config loaders, dynamic module imports, message serialization |
| `brainbot_command_service` | Remote command service orchestrating teleop/AI providers      |
| `brainbot_control_service` | Robot-side loop interfacing with motors and camera streamer   |
| `brainbot_mode_dispatcher` | CLI dispatcher (swappable for other event sources)            |
| `brainbot_teleop_server`   | Exposes local teleop devices or AR controllers via ZMQ        |
| `brainbot_webviz`          | HTTP dashboard for command history, mode, and camera preview  |
| `brainbot/scripts`         | Launch helpers for the remote robot host (`remote/`) and hub host (`hub/`) |
| `brainbot_service_manager` | Shared ZeroMQ service manager used by hub/remote hosts                  |

## Typical Workflow

Brainbot operates with two machines (**or on the same machine if you wish**):

- **remote_host** – runs the command service, robot controller, and optional policy server (typically the edge computer on the robot).
- **hub_host** – launches teleoperation and data-collection servers on demand.

### 1. Start the service manager on `hub_host`
```bash
python scripts/hub/run_all.py
```
The manager idles until the robot requests a teleop/data service. Logs report when subprocesses start/stop.
Edit `brainbot/scripts/hub/hub_manager.yaml` to add or modify the services that can be launched (each entry maps to a teleop or data server configuration).

### 2. Launch Brainbot on `remote_host`
```bash
# Full stack (command service + robot controller + optional preview camera bridge)
export HUB_HOST=<hub_host_ip>
python scripts/remote/run_all.py

# or start components individually
export HUB_HOST=<hub_host_ip>
python scripts/remote/run_thor_command.py --config scripts/remote/thor_command.yaml
python scripts/remote/run_thor_robot.py --config scripts/remote/thor_robot.yaml --no-calibrate
```

> All commands are intended to be executed from the repository root (`brainbot` directory) on the respective host.
> If `HUB_HOST` is not set, the command service assumes `127.0.0.1` for any unresolved host strings.

If any teleop/data server is configured with `host: 127.0.0.1` (e.g., the AR teleop running on `remote_host`), start a service manager locally as well:

```bash
python scripts/hub/run_all.py  # run on remote_host when local services are required
```

The service manager itself lives in `brainbot_service_manager/` and exposes a simple ZeroMQ API for starting, stopping, and listing managed services. Both the hub and remote entry points reuse this module, so extending or debugging the lifecycle logic requires changing it in a single place.

### 3. (Optional) Start the policy server
If you are using GR00T or another inference service, start it on `remote_host` before issuing inference mode requests (default ZMQ endpoint `127.0.0.1:5555`).

### Mode Control & Visualization

### Socket Mode Commands

Launch the services with a UNIX socket dispatcher (examples assume `/tmp/brainbot_modesock`):

```bash
python scripts/remote/run_all.py --mode-socket /tmp/brainbot_modesock
# …or the command service only
python scripts/remote/run_thor_command.py --mode-socket /tmp/brainbot_modesock
```

Send mode changes from another terminal using the helper script:

```bash
# Switch teleop provider
python scripts/remote/send_mode_command.py teleop leader
python scripts/remote/send_mode_command.py teleop joycon
python scripts/remote/send_mode_command.py teleop gamepad
python scripts/remote/send_mode_command.py teleop ar

# AI instructions
python scripts/remote/send_mode_command.py infer "Pick up the block"
python scripts/remote/send_mode_command.py infer "Open the shelf"

# Idle / shutdown
python scripts/remote/send_mode_command.py idle
python scripts/remote/send_mode_command.py shutdown

# Data collection helpers
python scripts/remote/send_mode_command.py data start
python scripts/remote/send_mode_command.py data resume
python scripts/remote/send_mode_command.py data next
python scripts/remote/send_mode_command.py data stop
python scripts/remote/send_mode_command.py data rerecord

# Raw JSON fallback when you need a custom payload
python scripts/remote/send_mode_command.py raw '{"teleop":"leader"}'
```

Responses from the command service are printed to stdout (`OK`, errors, or blank on success).

- Send JSON commands in runtime on the edge machine to swap providers:
  - `{"teleop": "leader"}`
  - `{"teleop": "ar"}`
  - `{"teleop": "joycon"}`
  - `{"teleop": "gamepad"}`
  - `{"infer": "Pick up the block using the left arm and transfer!"}`
  - `{"infer": "Open the shelf"}`
  - `{"data": {"command": "start"}}` (start data recording mode)
  - `{"idle": ""}`
  - `{"shutdown": ""}` (gracefully shuts down robot controller, then the hub)
- View live command traces, numeric history, and camera previews at `http://<edge-ip>:8080/`.
- `run_thor_preview.py` keeps WebViz and camera streams active before the robot agent starts.

### Data Recording Mode

Brainbot includes a data collection mode that integrates with LeRobot datasets for recording robot demonstrations. This mode captures synchronized robot observations, actions, and video streams for training imitation learning policies.

#### Configuration

Add a data recording configuration to your command service YAML.

#### Usage

1. **Start Data Recording Mode:**
   ```json
   {"data": {"command": "start"}}
   ```

2. **Control Recording Session:**
   - **Begin Recording:** `{"data": {"command": "resume"}}` or `{"data": {"command": "go"}}`
   - **Pause Recording:** `{"data": {"command": "reset"}}`
   - **Save Current Episode:** `{"data": {"command": "next"}}` or `{"data": {"command": "skip"}}`
   - **Stop Data Collection:** `{"data": {"command": "stop"}}` or `{"data": {"command": "end"}}`
   - **Re-record Episode:** `{"data": {"command": "rerecord"}}` or `{"data": {"command": "redo"}}`
   - The helper script automatically switches into the data mode slot before issuing the control command:
     ```bash
     python scripts/remote/send_mode_command.py data start
     python scripts/remote/send_mode_command.py data next
     ```

3. **Typical Recording Workflow:**
   ```bash
   # Start data mode
   {"data": {"command": "start"}}
   
   # Set up environment and begin recording
   {"data": {"command": "resume"}}
   
   # Perform demonstration...
   
   # Save episode and move to next
   {"data": {"command": "next"}}
   
   # Repeat for multiple episodes...
   
   # Stop when done
   {"data": {"command": "stop"}}
   ```

## Config Examples

`brainbot/scripts/remote/thor_command.yaml`:
```yaml
teleops:
  leader:
   mode: remote
    host: ${HUB_HOST}
    port: 7001
    timeout_ms: 1000
    config: ../hub/teleop_server.yaml
  joycon:
   mode: remote
    host: ${HUB_HOST}
    port: 7002
    timeout_ms: 1000
    config: ../hub/joycon.yaml
  gamepad:
   mode: remote
    host: ${HUB_HOST}
    port: 7003
    timeout_ms: 1000
    config: ../hub/gamepad.yaml
  ar:
    mode: remote
    host: 127.0.0.1
    port: 7004
    timeout_ms: 1000
    config: ../hub/ar_teleop.yaml
ai:
  host: 172.17.0.3
  port: 5555
  timeout_ms: 10000
  modality_config_path: scripts/remote/xlerobot_modality.json
  camera_keys: [left, right, top]
  state_keys:
    - left_shoulder_pan.pos
    - left_shoulder_lift.pos
    - left_elbow_flex.pos
    - left_wrist_flex.pos
    - left_wrist_roll.pos
    - left_gripper.pos
    - right_shoulder_pan.pos
    - right_shoulder_lift.pos
    - right_elbow_flex.pos
    - right_wrist_flex.pos
    - right_wrist_roll.pos
    - right_gripper.pos
    - x.vel
    - y.vel
    - theta.vel
    - mount_pan.pos
    - mount_tilt.pos
network:
  host: 127.0.0.1
  port: 6000
webviz:
  host: 0.0.0.0
  port: 8080
camera_stream:
  host: 127.0.0.1
  port: 7005
  quality: 70
  sources:
    - name: left
      path: robot.cameras.left
      fps: 15
    - name: right
      path: robot.cameras.right
      fps: 15
    - name: top
      path: robot.cameras.top
      fps: 15
```

RobotControlService now switches to numeric-only observations when a teleop provider is active and brings back preprocessed camera frames automatically while AI mode is running, keeping both workflows responsive.
- Median + low-pass action filtering smooths teleop and inference commands before they reach the robot.

Adjust the modality path, camera keys, and state keys so they match the GR00T build you deploy (add or remove base/mount keys as needed).

Set the `HUB_HOST` environment variable on `remote_host` to the reachable IP address of your hub host. If unset, the YAML defaults fall back to `192.168.22.171`.

`brainbot/scripts/hub/leader_teleop.yaml`:
```yaml
teleop:
  mode: local
  config:
    type: xlerobot_leader_gamepad
    id: leader
    arms:
      left_arm_port: /dev/ttyACM0
      right_arm_port: /dev/ttyACM1
    base:
      joystick_index: 0
      max_speed_mps: 0.8
      deadzone: 0.15
      yaw_speed_deg: 45
    mount:
      joystick_index: 0
      deadzone: 0.15
      max_pan_speed_dps: 60.0
      max_tilt_speed_dps: 45.0
network:
  host: 0.0.0.0
  port: 7001
```

`brainbot/scripts/remote/thor_robot.yaml`:
```yaml
robot:
  type: xlerobot
  id: follower
  arms:
    left_arm_port: /dev/ttyACM2
    right_arm_port: /dev/ttyACM3
  base:
    port: /dev/ttyACM4
    wheel_radius_m: 0.05
    base_radius_m: 0.125
  mount: {}
  cameras:
    left:  {type: opencv, index_or_path: 8, width: 640, height: 480, fps: 15, enable_mjpeg: true}
    right: {type: opencv, index_or_path: 6, width: 640, height: 480, fps: 15, enable_mjpeg: true}
    top:   {type: opencv, index_or_path: 4, width: 640, height: 480, fps: 15, enable_mjpeg: true}
network:
  host: 127.0.0.1
  port: 6000
loop_hz: 40
max_missed_actions: 2
calibrate_on_start: true
observation_adapter: identity        # start in full mode; dynamic switching keeps teleop fast
observation_preprocess:
  target_height: 224
  target_width: 224
  interpolation: linear
action_filter:
  type: median
  window_size: 5
  blend_alpha: 0.3
camera_stream:
  host: 0.0.0.0
  port: 7005
  quality: 70
  sources:
    - name: left
      path: robot.cameras.left
      fps: 15
    - name: right
      path: robot.cameras.right
      fps: 15
    - name: top
      path: robot.cameras.top
      fps: 15
```

## Camera Streaming Protocol
- **Capture:** LeRobot still acquires frames; `enable_mjpeg: true` requests MJPEG from cameras to save USB bandwidth when supported.
- **Publisher:** `brainbot_control_service` keeps raw arrays for GR00T, and asynchronously JPEG-encodes configured sources for visualization. Frames publish via ZMQ `PUB` (`tcp://host:port`, topic = camera name). Message format is MsgPack with `camera`, `timestamp`, `encoding`, `width`, `height`, `quality`, and JPEG bytes.
- **Consumers:** Unity, ROS bridges, WebViz, or custom tools subscribe via ZMQ `SUB`. WebViz renders previews automatically.

## Notes
- Robot/teleop/camera modules auto-import at load time so YAML configs “just work”.
- Remote teleops are pinged and keep socket timeouts, preventing the control loop from blocking.
- Switching out of AI mode clears the last instruction; idle is always available as a safe fallback.

With these pieces, you can teleoperate, stream multiple cameras, run GR00T policies, and monitor everything across the remote_host, hub_host, and auxiliary AR controllers—all within Brainbot’s modular stack.
