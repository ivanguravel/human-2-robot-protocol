# Human 2 Robot Protocol

The A2R protocol is organized into 5 layers, each with a clear responsibility:

```
┌────────────────────────────────────────────────────────────────────────┐
│  Layer 4: Intent                                                       │
│  Natural language input in any human language                          │
│  "walk 3 steps forward and turn left"                                  │
├────────────────────────────────────────────────────────────────────────┤
│  Layer 3: Skill                                                        │
│  AI agent + mapping rules (.cursor/skills/go2-control/SKILL.md)        │
│  Converts intent → structured commands (1 step = 0.3m, left = +90°)   │
├────────────────────────────────────────────────────────────────────────┤
│  Layer 2: Command                                                      │
│  Structured CLI with JSON contract                                     │
│  go2_cli.py walk --forward 0.9  →  {"status": "ok", ...}              │
├────────────────────────────────────────────────────────────────────────┤
│  Layer 1: Transport                                                    │
│  Backend-specific adapter (swappable)                                  │
│  SimClient (docker exec → ros2)  |  Go2Client (unitree SDK → DDS)     │
├────────────────────────────────────────────────────────────────────────┤
│  Layer 0: Hardware                                                     │
│  Physical actuators or physics engine                                  │
│  Unitree Go2 motors  |  Gazebo Sim rigid-body dynamics                 │
└────────────────────────────────────────────────────────────────────────┘
```

The key design principle: **each layer only knows about the layer directly below it.** The AI agent never touches DDS or ROS — it only executes CLI commands and reads JSON. The CLI never knows whether it's talking to a real robot or a simulator.

## Quick Start (Simulation)

### 1. Start the simulator

```bash
cd docker
docker compose -f compose.yml build simulator
docker compose -f compose.yml up simulator -d
```

Wait ~30 seconds for ROS nodes to initialize.

Open **http://localhost:8080/vnc.html** to see the Gazebo 3D view.

### 2. Control via CLI

```bash
# Check robot is alive
python3 go2_real/go2_cli.py --sim status

# Walk forward 1 meter
python3 go2_real/go2_cli.py --sim walk --forward 1.0

# Rotate 90 degrees left
python3 go2_real/go2_cli.py --sim rotate 90

# Sit down
python3 go2_real/go2_cli.py --sim sit
```

### 3. Control via Cursor Agent

Just write in the Cursor chat (Agent mode):

> 3 steps forward and 2 steps left

The `go2-control` skill automatically translates natural language to CLI commands.

**Convention:** 1 step = 0.3 meters.

### 4. Voice Control (Speech-to-Text)

Control the robot with your voice — speak in Russian or English, the faster-whisper STT server transcribes your speech and the Cursor Agent CLI executes the command.

```bash
# Start whisper + simulator
docker compose -f docker/compose.yml up simulator whisper -d

# Install Cursor Agent CLI (one time)
curl https://cursor.com/install -fsS | bash

# Install voice dependencies (one time, on host)
pip install -r go2_real/requirements-voice.txt

# Run voice control
python3 go2_real/voice_control.py
```

Usage:
1. Press **Enter** to start recording
2. Say your command (e.g. "три шага вперёд и поверни налево")
3. Press **Enter** to stop recording
4. Whisper transcribes your speech
5. Cursor Agent interprets the command via go2-control skill
6. The robot moves in the simulator

> First start of the whisper container downloads the model (~500 MB) — takes 1-2 min, then cached.
> Total latency: speech → robot moves in ~22 seconds (sim) or ~7 seconds (real robot).

Environment variables (optional):
- `WHISPER_URL` — STT endpoint (default: `http://localhost:9000/v1/audio/transcriptions`)
- `WHISPER_API_KEY` — Bearer token if auth is enabled (disabled by default)
- `GO2_WORKSPACE` — project root (auto-detected)

## Quick Start (Real Robot)

### Prerequisites

- Unitree Go2 EDU connected via Ethernet
- PC on `192.168.123.x` subnet (robot is at `192.168.123.161`)

### Setup

```bash
cd go2_real
pip install -r requirements.txt
export GO2_IFACE=eth0  # your Ethernet interface
```

### Control

```bash
python3 go2_real/go2_cli.py status
python3 go2_real/go2_cli.py stand
python3 go2_real/go2_cli.py walk --forward 1.0
python3 go2_real/go2_cli.py rotate -90
python3 go2_real/go2_cli.py sit
```

Or via Cursor Agent — same natural language commands, without `--sim`.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  User (voice / text)                                    │
└────────────────────────────┬────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────┐
│  Cursor Agent + .cursor/skills/go2-control/SKILL.md     │
│  (parses intent, maps to CLI commands)                  │
└────────────────────────────┬────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────┐
│  go2_real/go2_cli.py                                    │
│  walk / rotate / stand / sit / stop / status            │
│  JSON output, exit code 0/1                             │
└──────────┬─────────────────────────────┬────────────────┘
           │ --sim                       │ (default)
           ▼                             ▼
┌─────────────────────┐    ┌──────────────────────────────┐
│  SimClient           │    │  Go2Client                   │
│  docker exec →       │    │  unitree_sdk2_python →       │
│  ros2 topic pub     │    │  SportClient.Move()          │
│  (open-loop, timed) │    │  (closed-loop, odometry)     │
└──────────┬──────────┘    └──────────────┬───────────────┘
           │                              │
           ▼                              ▼
┌─────────────────────┐    ┌──────────────────────────────┐
│  Gazebo Sim (Docker) │    │  Real Unitree Go2            │
│  ROS 2 Jazzy         │    │  DDS over Ethernet           │
└─────────────────────┘    └──────────────────────────────┘
```

## CLI Reference

```
python3 go2_real/go2_cli.py [--sim | --dry-run] [--container NAME] [--iface IF] COMMAND

Commands:
  walk     --forward M --right M    Move by meters
  rotate   DEGREES                  Rotate (positive=left, negative=right)
  stand                             Stand up / activate trot
  sit                               Sit down
  stop                              Emergency stop
  status                            Get telemetry

Modes:
  (default)    Real robot via unitree_sdk2_python
  --sim        Gazebo simulation via docker exec
  --dry-run    Print plan without connecting

Environment:
  GO2_MODE=sim          Same as --sim
  GO2_IFACE=eth0        Network interface for real robot
  GO2_CONTAINER=name    Docker container name for sim
```

## Safety Limits

| Parameter | Sim | Real |
|-----------|-----|------|
| Max distance / command | 3.0 m | 3.0 m |
| Max speed | 0.035 m/s | 0.5 m/s |
| Max yaw rate | 0.5 rad/s | 1.0 rad/s |
| Timeout | 60 s | 30 s |

## Project Structure

```
go2_real/                ← Agent 2 Robot Protocol
├── go2_cli.py           CLI (unified interface)
├── go2_client.py        Real robot backend (unitree SDK)
├── sim_client.py        Simulation backend (docker exec)
├── voice_control.py     Push-to-talk voice control script
├── requirements.txt     SDK dependencies
├── requirements-voice.txt  Voice control dependencies
└── README.md            Setup guide

.cursor/skills/go2-control/
└── SKILL.md           Cursor skill (NL → CLI mapping)

docker/                Docker setup for Gazebo sim + STT
├── compose.yml        simulator / simulator-headless / whisper services
├── Dockerfile         ROS 2 Jazzy + Gazebo + Nav2
├── supervisord.conf   Xvfb + VNC + noVNC + simulator
└── cyclonedds.xml     DDS config

presentation/          Conference talk generator
├── generate_pptx.py   Builds the PPTX presentation
├── requirements.txt   python-pptx
└── a2r_protocol.pptx  Generated slides (20 slides)

gazebo_sim/            ROS 2 simulation package
go2_description/       Unitree Go2 URDF/meshes
go1_description/       Unitree Go1 URDF/meshes
quadropted_controller/ Python gait controller (sim only)
quadropted_msgs/       Custom ROS 2 messages
```

## Credits

- [unitree_sdk2_python](https://github.com/unitreerobotics/unitree_sdk2_python) — official Unitree Go2 SDK
- [abutalipovvv/go_sim_py](https://github.com/abutalipovvv/go_sim_py) — original simulation
- [mike4192/spotMicro](https://github.com/mike4192/spotMicro) — gait algorithms
- [lnotspotl](https://github.com/lnotspotl) — inverse kinematics

## TODO

- [x] Add speech-to-text layer (voice → agent → robot)
- [ ] Closed-loop control in simulation (use odometry feedback)
- [ ] ROS 2 bridge node for Nav2 integration with real robot
- [ ] Calibrate sim speed constants for better distance accuracy
- [ ] Continuous voice mode (wake word instead of push-to-talk)
- [ ] Vision layer: camera feed → spatial commands ("go to the red box")
