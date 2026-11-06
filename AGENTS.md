# AGENTS.md

Guidance for AI coding agents working in this repository.

## What this repo is

**Human 2 Robot Protocol (H2R)** — natural-language / voice control of a Unitree Go2 quadruped,
both in a Gazebo simulation (Docker) and on the real robot. See `README.md` for the full picture.

The protocol is layered; **each layer only talks to the layer directly below it**:

| Layer | Where | Responsibility |
|-------|-------|----------------|
| 4 Intent | user text / `go2_real/voice_control.py` | Natural language in, any language |
| 3 Skill | `.cursor/skills/go2-control/SKILL.md` | Maps intent → CLI commands |
| 2 Command | `go2_real/go2_cli.py` | Stable CLI, JSON on stdout, exit code 0/1 |
| 1 Transport | `go2_real/sim_client.py`, `go2_real/go2_client.py` | Swappable backends with the same interface |
| 0 Hardware | Gazebo (`gazebo_sim/`, `docker/`) or real Go2 | Physics / motors |

Never let an upper layer reach past the one below it (e.g. the skill must not call `ros2` or
`docker exec` directly — it only runs `go2_cli.py`).

## Repository map

- `go2_real/` — the A2R code (plain Python 3, **no ROS dependency** on the host).
- `.cursor/skills/go2-control/SKILL.md` — NL → CLI mapping, safety rules, conventions.
- `docker/` — `compose.yml` (services `simulator`, `simulator-headless`, `whisper`), `Dockerfile`
  (ROS 2 Jazzy + Gazebo + Nav2), `supervisord.conf` (Xvfb + x11vnc + noVNC + sim launch).
- `gazebo_sim/` — ROS 2 package: launch files, worlds, Nav2 config, `config/robots.yaml`.
- `quadropted_controller/`, `quadropted_msgs/` — gait controller and messages (sim only, runs in the container).
- `go1_description/`, `go2_description/` — URDF + meshes. Treat as vendored assets.
- `presentation/` — python-pptx generators for talk / pitch decks (not part of the robot stack).
- `FLEET_DEMO_PLAN.md` — roadmap for multi-robot / drone work (not implemented yet).

## Running things

```bash
# Start the simulator (GUI via noVNC) and the STT server
docker compose -f docker/compose.yml up simulator whisper -d
# wait ~30-40 s for ROS nodes, then view http://localhost:8080/vnc.html

# Smoke test: status → walk 0.6 m forward → walk 0.3 m left
bash .run_sim_test.sh

# Single commands
python3 go2_real/go2_cli.py --sim status
python3 go2_real/go2_cli.py --sim walk --forward 0.9
python3 go2_real/go2_cli.py --dry-run rotate 90     # no robot needed

# Voice control (needs whisper + Cursor Agent CLI logged in)
python3 go2_real/voice_control.py
```

There is no unit-test suite. After changing `go2_real/`, verify with `--dry-run` for every touched
command and, if the simulator is up, with `.run_sim_test.sh`.

## Conventions

- **CLI contract is the API.** Every `go2_cli.py` command prints a single JSON object and exits
  `0` on success, `1` on failure. Don't print anything else to stdout; keep existing keys stable
  (`status`, `mode`, `distance_moved`, `elapsed_s`, …).
- **Units:** meters and degrees at the CLI; 1 step = 0.3 m; positive rotation = left (CCW);
  `walk --right` with a negative value moves left.
- **Backend parity:** a new command must be implemented in `SimClient` **and** `Go2Client` (or fail
  explicitly with a JSON error), wired into `go2_cli.py`, and added to the mapping table in
  `SKILL.md` and the CLI reference in `README.md`. `dance` is the reference example.
- **Safety limits** live as module constants at the top of `go2_client.py` / `sim_client.py`
  (`MAX_DISTANCE_PER_CMD = 3.0`, speed / yaw caps, timeouts). Don't raise them without being asked.
- Sim motion is open-loop and timed (`SIM_SPEED = 0.035` m/s), so sim moves are slow — a 0.9 m walk
  takes ~30 s. Use generous timeouts when running commands.
- Sim robot namespace is hard-coded (`ROBOT_NAMESPACE = "robot1"` in `sim_client.py`); default
  container is `simulator-gui` (override with `--container` / `GO2_CONTAINER`).
- Configuration via env vars, not new config files: `GO2_MODE`, `GO2_IFACE`, `GO2_CONTAINER`,
  `WHISPER_URL`, `WHISPER_API_KEY`, `GO2_WORKSPACE`.
- Match the existing style: small functions, module-level constants, type hints where already used,
  sparse comments.

## Gotchas

- **Docker on this machine runs unrelated containers** from other projects. Only start/stop/restart
  `simulator-gui` and `whisper` — never `docker compose down` everything or prune.
- **Black noVNC screen** after container start = Xvfb/Gazebo startup race in supervisord
  (`simulator` program has `autorestart=false`). Fix: `docker restart simulator-gui`.
- **Whisper:** `WHISPER_API_KEY=` is intentionally empty in `compose.yml` to disable auth. If port
  9000 is busy, the container is created without a port binding — recreate it with
  `docker compose -f docker/compose.yml up whisper -d --force-recreate`.
- **Cursor Agent CLI** is invoked as `agent -p --trust --yolo --workspace <repo> "[sim mode] …"`.
  `Authentication required` means the token expired → `agent login`.
- `__pycache__/*.pyc` files under `quadropted_controller/` are tracked in git by mistake; don't
  commit changes to them.
- `.run_sim_test.sh` / `.run_go2_cmds.sh` contain an absolute local path — keep that in mind before
  publishing them.
- Large binaries (meshes, media) are stored with git-lfs in the published copy of the repo; don't
  add big files (zips, videos) to git.

## Real robot

Commands without `--sim` drive a physical robot over Ethernet (`192.168.123.161`). Do not run them
unless the user explicitly asks for the real robot. Always run `status` first and stop on the first
non-zero exit code.
