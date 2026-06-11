# fairinogale

**Arkitekt node for Fairino collaborative robot arms.**  
Exposes robot pick-and-place routines (liquidhandler ↔ microscope ↔ pickup station) as callable Arkitekt node so they can be orchestrated from any Arkitekt workflow.

---

## Features

- Connect to a Fairino robot over TCP/IP and expose its motion commands to [Arkitekt](https://arkitekt.live)
- Four registered Arkitekt nodes out of the box:
  - `pick_up_frame` — pick samples from the staining frame
  - `release_at_frame` — place samples onto the staining frame
  - `pick_up_opentrons` — pick samples from an Opentrons liquid handler
  - `release_at_opentrons` — release samples into an Opentrons liquid handler
- JSON-based teach-point system: record poses once, replay them reliably
- Automatic speed reduction for "danger" and "gripper" waypoints
- MockRPC fallback for development without physical hardware
- Cross-platform SDK loader (Windows / Linux); override path via `FAIRINO_SDK_DIR`

---

## Requirements

- Python ≥ 3.12
- [uv](https://github.com/astral-sh/uv) (recommended) or pip
- Fairino Python SDK (bundled under `fairino-python-sdk/`)
- A running [Arkitekt](https://arkitekt.live) server (local or `go.arkitekt.live`)
- Fairino robot reachable at its configured IP address (default `192.168.50.200`)

---

## Installation

```bash
# 1. Clone the repo
git clone https://github.com/<your-org>/fairinogale.git
cd fairinogale

# 2. Install dependencies with uv
uv sync

# 3. Copy and fill in the environment file
cp .env.example .env
```

If you don't have `uv`:

```bash
pip install arkitekt-next
```

---

## Configuration

Create a `.env` file in the project root (see up in the Installation section, or export these variables in your shell):

| Variable | Default | Description |
|---|---|---|
| `ARKITEKT_APPNAME` | — | **Required.** App identifier registered in Arkitekt (e.g. `arkirino`) |
| `ARKITEKT_URL` | `go.arkitekt.live` | URL of your Arkitekt server |
| `REDEEM_TOKEN` | — | One-time redeem token for first-time app registration |
| `FAIRINO_SDK_DIR` | auto-detected | Absolute path to the `fairino` SDK folder (overrides auto-detection) |

Example `.env`:

```env
ARKITEKT_APPNAME=arkirino
ARKITEKT_URL=go.arkitekt.live
REDEEM_TOKEN=your_token_here
# Optional: override SDK path for CI or containers
# FAIRINO_SDK_DIR=/home/pi/arkirino/fairino-python-sdk/linux/fairino
```

Robot and gripper settings are configured directly in [app.py](app.py):

```python
ROBOT_IP = "192.168.1.2"       # IP address of the Fairino robot
SPEED    = 30                  # Default joint speed (%)
```

---

## Usage

### Start the app

```bash
uv run python app.py
```

The app connects to the robot, initializes the gripper, and registers itself with Arkitekt. Once running, the four motion nodes appear in the Arkitekt UI and can be wired into any workflow.

### Run without hardware (MockRPC)

If the Fairino SDK is not installed or the robot is unreachable, the app falls back to `MockRPC` automatically.

---

## Registered Arkitekt Nodes

| Node | Parameter | Description |
|---|---|---|
| `init_robot_and_gripper` | — | Initialize robot and gripper before a run |
| `pick_up_frame` | `sample` | Pick samples from the staining frame |
| `release_at_frame` | `sample` | Place samples onto the staining frame |
| `pick_up_opentrons` | `sample` | Pick samples from the Opentrons |
| `release_at_opentrons` | `sample` | Release samples into the Opentrons |

All motion nodes accept optional `speed`, `acceleration`, and `dangerSpeed` parameters.

---

## Examples

### Trigger a pick-and-place from Python

```python
from app import init, pick_up_frame, release_at_opentrons

init()
pick_up_frame(sample="Box1_A1")
release_at_opentrons(sample="Box1_A1")
```

### Run the standalone test script for testing

```bash
uv run python testing/control_robotarm.py
```

Edit the `__main__` block at the bottom to choose which movement to execute.

---

## Teach Points

Teach points are robot poses stored as JSON files under `control-points/`. Each entry is a named waypoint with 20 values (joint angles + Cartesian coordinates + metadata).

### Naming conventions

The point name suffix controls automatic speed and gripper behaviour:

| Suffix | Effect |
|---|---|
| `_danger` | Speed reduced to `danger_speed` |
| `_open-gripper` | Move to pose, then **open** gripper |
| `_close-gripper` | Move to pose, then **close** gripper |
| _(none)_ | Normal speed, no gripper action |

### Recording new teach points

1. Jog the robot to the desired pose using the teach pendant.
2. Name the point in the Fairino teach interface following the suffix conventions above.
3. Run the capture script, editing `teach_point_names` to match your point names:

```bash
uv run python testing/get_points.py
```

Points are saved to `control-points/<your_file>.json` and can be referenced directly by the motion functions in `app.py`.

---

## fairino-python-sdk

Introduction
------------
This is a Python language SDK library specially designed for fairino collaborative robots.

Documentation
-------------
Please see [Python SDK](https://fair-documentation.readthedocs.io/en/latest/SDKManual/python_intro.html).


---

## Project Structure

```
fairinogale/
├── app.py                    # Main Arkitekt app — registered nodes live here
├── pyproject.toml
├── .env                      # Local config (not committed)
├── control-points/           # Teach-point JSON files per movement
│   ├── pick_up_frame.json
│   └── pick_up_opentrons_Box1_A1.json
├── testing/
│   ├── control_robotarm.py   # Standalone movement test script
│   └── get_points.py         # Utility to capture teach points from the robot
└── fairino-python-sdk/       # Vendored Fairino SDK (Windows + Linux)
```
