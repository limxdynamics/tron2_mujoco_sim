<!--
  SPDX-FileCopyrightText: 2024-2026 LimX Dynamics Technology Co., Ltd.
  SPDX-License-Identifier: Apache-2.0
-->

<p align="center">
  <img src="doc/banner.png" alt="TRON2 MuJoCo Simulator: the DA, SF, DASF, WF and DACH TRON2A models rendered from robot-description" width="100%">
</p>

<p align="center">
  <b>English</b> | <a href="README_zh-CN.md">中文</a>
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-Apache--2.0-2f6feb" alt="License: Apache-2.0"></a>
  <img src="https://img.shields.io/badge/python-%E2%89%A53.10-2f6feb" alt="Python >= 3.10">
  <img src="https://img.shields.io/badge/MuJoCo-%E2%89%A53.2.2-2f6feb" alt="MuJoCo >= 3.2.2">
  <img src="https://img.shields.io/badge/limxsdk-4.0.2-2f6feb" alt="limxsdk 4.0.2">
  <a href="https://github.com/limxdynamics/limxdynamics/blob/main/doc/wechat_qr.png"><img src="https://img.shields.io/badge/WeChat-Developer%20Community-07C160?logo=wechat&logoColor=white" alt="WeChat Community"></a>
</p>

# tron2-mujoco-sim

> **Part of the TRON2 open-source ecosystem.** Where to start and the other TRON2 repositories: [TRON2 quick start](https://github.com/limxdynamics/limxdynamics/blob/main/doc/quickstart.md#tron2).

MuJoCo simulator for the TRON2 robot family. It bridges the LimX low-level SDK
(`RobotCmd` / `RobotState` carrying `q` / `dq` / `tau` / `Kp` / `Kd`, plus
`ImuData` and gripper messages) to `mujoco.MjData`, so the same controller wire
format that drives a physical robot can be exercised against a simulated one.

A robot variant is declared, not hard-coded: it is a set of joint channels plus
optional modules, assembled in `tron2_sim/variants/`. Every channel of every
supported variant runs over the SDK's `*ForSim` simulator-side API — there is no
second messaging stack.

> **Scope:** this is the simulator side only; it runs no policy of its own.
> Connect your controller over `limxsdk`, for example
> [tron2_rl_deploy_python](https://github.com/limxdynamics/tron2_rl_deploy_python).
> The simulator mirrors the hardware interface; it is not a safety net
> (see [`SECURITY.md`](SECURITY.md)).

## Highlights

- **The robot's own wire format.** Controllers send `RobotCmd` and read
  `RobotState` / `ImuData` through `limxsdk`, exactly as on hardware; only the
  endpoint changes.
- **Ten robot types, one entry point.** SF, WF, DA, DACH and DASF, each as
  TRON2A and TRON2B, selected with `ROBOT_TYPE`.
- **Joints mapped by name.** Wire order is resolved by joint name, so a model
  whose XML declares actuators in a different order still matches the robot.
- **Steady 1 kHz physics.** Physics and state publishing run on their own
  thread; the viewer renders from a separate `MjData` and never stalls them.
- **Headless for CI and remote machines.** `--headless --duration 30` runs the
  same physics and I/O without graphics.
- **DACH 2F gripper included.** A linkage gripper module with its calibration
  in `tron2_sim/config/gripper_config.yaml`.

## Model lineup

| | Model | Description | DoF | Formats |
|:---:|---|---|:---:|---|
| <img src="doc/model_sf.png" alt="SF_TRON2A" width="320"> | **SF**<br>`SF_TRON2A`<br>`SF_TRON2B` | Sole-foot biped. One SDK channel: `/motor/cmd\|state` + `/ImuData`. | 10 | MJCF · URDF · USD |
| <img src="doc/model_wf.png" alt="WF_TRON2A" width="320"> | **WF**<br>`WF_TRON2A`<br>`WF_TRON2B` | Wheel-foot biped: 8 leg joints and 2 wheels on one channel. | 10 | MJCF · URDF · USD |
| <img src="doc/model_da.png" alt="DA_TRON2A" width="320"> | **DA**<br>`DA_TRON2A`<br>`DA_TRON2B` | Dual arm on a stand, 7 joints per arm. | 14 | MJCF · URDF · USD |
| <img src="doc/model_dach.png" alt="DACH_TRON2A" width="320"> | **DACH**<br>`DACH_TRON2A`<br>`DACH_TRON2B` | Dual arm + 2-DOF head. Loads `robot_grasper.xml` with the 2F gripper by default; `--no-grasper` for `robot.xml`. | 16 | MJCF · URDF · USD |
| <img src="doc/model_dasf.png" alt="DASF_TRON2A" width="320"> | **DASF**<br>`DASF_TRON2A`<br>`DASF_TRON2B` | Humanoid composite: SF legs + DACH arms and head, on two channels (lower 10, upper 16). | 26 | MJCF · URDF · USD |

Models come from the [`robot-description`](https://github.com/limxdynamics/tron2-robot-description)
submodule; the simulator loads the MJCF, and TRON2A also ships Xacro sources.
Rendered in MuJoCo with one camera and light setup; DASF is shown with its
shoulders rolled down, the others in their zero pose.

## Demos

Recorded from this simulator, one per variant.

| `SF_TRON2A` | `WF_TRON2A` |
|:---:|:---:|
| <img src="doc/gif/SF_TRON2A.gif" alt="SF_TRON2A demo" width="100%"> | <img src="doc/gif/WF_TRON2A.gif" alt="WF_TRON2A demo" width="100%"> |
| **`DACH_TRON2A`** | **`DASF_TRON2A`** |
| <img src="doc/gif/DACH_TRON2A.gif" alt="DACH_TRON2A demo" width="100%"> | <img src="doc/gif/DASF_TRON2A.gif" alt="DASF_TRON2A demo" width="100%"> |

See [`doc/gif/README.md`](doc/gif/README.md) for the media rules CI enforces.

## How it works

<p align="center">
  <img src="doc/architecture_en.png" alt="Controller to limxsdk to SdkBus to JointChannel to the 1 kHz physics thread, with state and IMU published back; a separate render thread drives the viewer" width="100%">
</p>

`ROBOT_TYPE` picks a variant from `tron2_sim/variants/`, which declares its
channels and modules. Each physics tick applies the MIT law to the latest
command, steps MuJoCo by 1 ms, reads the joint and IMU state and publishes it
back through the SDK.

## Quick start

Load a robot in the viewer without ROS or a controller (needs
[uv](https://docs.astral.sh/uv/); the `robot-description` checkout is about
1.7 GB):

```bash
git clone https://github.com/limxdynamics/tron2_mujoco_sim.git
cd tron2_mujoco_sim
git submodule update --init --depth 1 robot-description limxsdk-lowlevel
uv sync --extra sdk
ROBOT_TYPE=DASF_TRON2A uv run simulator.py
```

Until a controller connects on `127.0.0.1` the joints get no torque, so legged
models sag to the floor; press **Space** to pause and pose joints with the
sliders. Start a controller such as
[tron2_rl_deploy_python](https://github.com/limxdynamics/tron2_rl_deploy_python)
to drive it. Details and the pip route are in
[Dependencies and deployment](#1-dependencies-and-deployment).

## 1. Dependencies and deployment

| Dependency | Version | Notes |
|---|---|---|
| Python | >= 3.10 | verified on 3.10 |
| `mujoco` | >= 3.2.2 | physics and passive viewer |
| `PyYAML` | >= 6.0 | reads `tron2_sim/config/gripper_config.yaml` |
| `limxsdk` | 4.0.2 | architecture-specific wheel inside the `limxsdk-lowlevel` submodule; installed separately, always with `--no-deps` |

Clone first, either way:

```bash
git clone --recurse-submodules https://github.com/limxdynamics/tron2_mujoco_sim.git
cd tron2_mujoco_sim
```

If you already cloned without submodules, run
`git submodule update --init --recursive` before continuing.

### Option A: uv (recommended)

[uv](https://docs.astral.sh/uv/) installs the right Python, resolves everything
from the committed `uv.lock`, and needs no manual virtualenv:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh   # install uv
uv sync --extra sdk                               # physics + SDK

export ROBOT_TYPE=SF_TRON2A
uv run simulator.py                               # with viewer
uv run simulator.py --headless                    # no graphics (CI / remote)
uv run simulator.py --headless --duration 30      # exit after 30 s

uv run simulator.py --headless --duration 30 --no-grasper  # DACH without the 2F gripper
```

`uv sync` requires the `limxsdk-lowlevel` submodule to be checked out, since
that is where the SDK wheel lives. A later plain `uv sync` (without
`--extra sdk`) removes `limxsdk` again.

### Option B: pip

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -U pip
pip install "mujoco>=3.2.2" "pyyaml>=6.0"

# SDK wheel matching your architecture. --no-deps is required: the wheel's own
# metadata pulls in onnxruntime/pygame/scipy/pandas and pins numpy<1.26.4.
pip install --no-deps limxsdk-lowlevel/python3/amd64/limxsdk-*.whl     # x86_64
# pip install --no-deps limxsdk-lowlevel/python3/aarch64/limxsdk-*.whl # aarch64

export ROBOT_TYPE=SF_TRON2A
python3 simulator.py                              # with viewer
python3 simulator.py --headless --duration 30
```

### Environment variables

| Variable | Default | Effect |
|---|---|---|
| `ROBOT_TYPE` | *(required)* | Selects the variant, e.g. `SF_TRON2A`. |
| `ROBOT_IP` | `127.0.0.1` | SDK endpoint. |
| `GRIPPER_CONFIG` | `tron2_sim/config/gripper_config.yaml` | Loads a different DACH gripper calibration file. |
| `DACH_GRASPER` | `1` | `0` loads `robot.xml` instead of `robot_grasper.xml`. Same as `--no-grasper`. |

## 2. Supported robot types

Ten types: five base names in a `TRON2A` and a `TRON2B` variant. Joint names and
wire order are identical across the two families, so a controller needs no
changes to talk to either.

| Base | TRON2A | TRON2B | Description |
|---|---|---|---|
| SF | `SF_TRON2A` | `SF_TRON2B` | Sole-foot biped (10 joints) |
| WF | `WF_TRON2A` | `WF_TRON2B` | Wheel-foot biped (10 joints) |
| DA | `DA_TRON2A` | `DA_TRON2B` | Dual arm (14 joints) |
| DACH | `DACH_TRON2A` | `DACH_TRON2B` | Dual arm + 2-DOF head (16 joints), optional 2F gripper |
| DASF | `DASF_TRON2A` | `DASF_TRON2B` | Humanoid composite: SF legs + DACH arms/head (26 joints) |

## 3. Keyboard controls (viewer mode)

- **Space** — pause / resume. Paused means manual mode: the Control sliders on
  the right pose joints directly. (The DACH 2F gripper idles in manual mode and
  tracks neither sliders nor commands.)
- **R** — reset the floating-base pose only; joint angles are kept. Fixed-base
  models print a notice and ignore it.
- **Backspace / Reset button** — full reset (using the selected keyframe, when
  one is selected).
- **Double-click to select, then Ctrl+drag** — apply a perturbation force.

### 3a. Gamepad control

For the controller-side gamepad workflow (SF/WF variants running the
sibling [`tron2_rl_deploy_python`](https://github.com/limxdynamics/tron2_rl_deploy_python)
deployment stack), initialize the
`robot-joystick` submodule and run the helper binary alongside the
controller:

```bash
./robot-joystick/robot-joystick
```

Default bindings:

- `L1 + Y` — switch to WALK.
- `L1 + X` — switch back to IDLE.
- `R1` — clear velocity commands.

## 4. Communication

<p align="center">
  <img src="doc/channels_en.png" alt="Channels, SDK topics and wire order for SF, WF, DA, DACH and DASF" width="100%">
</p>

**Wire-order contract** (cmd/state index order, identical on TRON2A and TRON2B):
the DACH head is pitch then yaw; the DASF head is **yaw then pitch**. Where a
model's XML declares actuators in a different order, the simulator absorbs it by
resolving joints by name — never by position.

Behavioural notes:

- **Commands on Centaur channels (DASF) should carry joint names**
  (`RobotCmd.motor_names`). The native layer validates them against the
  published state and rejects mismatches with
  `ERROR: Centaur ... RobotCmd does not match the corresponding RobotState`.
  The simulator always publishes state with real joint names.
- DACH 2F gripper calibration lives in `tron2_sim/config/gripper_config.yaml`.
- On DACH grasper models, `grasper_base_{L,R}_Joint_ctrl` are driven by the
  linkage rather than by a channel, so their `ctrl` stays 0 and the simulator
  prints an `INFO: unowned actuators` line at startup. This is expected.

## 5. Architecture

```
simulator.py            entry point: ROBOT_TYPE -> variant registry -> SimCore
tron2_sim/
  spec.py               wire order + variant specs (pure data)
  core.py               physics/render dual-thread, dual-MjData snapshots,
                        manual mode, resets
  channels.py           JointChannel: name-based joint map, MIT control law,
                        state/IMU publishing
  transports.py         SdkBus (limxsdk *ForSim) and the capability guard
  config/               YAML loading + the shipped gripper_config.yaml
  modules/              optional capabilities; dach_grasper (2F linkage gripper)
  variants/             one file per base name; __init__.py is the registry
doc/gif/                demo animations (see the gallery above)
```

Threading: the physics thread owns the physics `MjData`; the render thread owns
`render_data`. UI resets and drag forces are handed over through flags and
buffers, so `MjData` always has exactly one writer.

## 6. TRON2B versus TRON2A

<p align="center">
  <img src="doc/tron2a_vs_tron2b_en.png" alt="SF_TRON2A and SF_TRON2B side by side with knee limits, torque and base mass" width="100%">
</p>

Joint names, ordering and topics are byte-identical, so controller code needs no
changes. But 2B is a hardware revision: **policies and gains tuned for 2A are not
directly transferable.**

| Difference | TRON2A | TRON2B |
|---|---|---|
| Hip pitch/roll and knee torque | ±150 N·m | **±200 N·m** (armature updated to match) |
| Hip yaw, ankle pitch, elbow torque | ±60 / ±70 N·m | **±70 N·m** |
| Wrist and head torque | ±20 N·m | **±15 N·m** |
| `base_Link` mass (SF/WF/DA/DACH) | 12.57 kg | **13.4 kg** (CoM and inertia updated) |
| SF knee / hip-yaw limits | `knee: [-2.618, 0.262]` | **sign-flipped** `knee: [-0.262, 2.618]`, link geometry mirrored |
| DA / DACH base | floating (near-rigid freejoint) | **fixed** (no freejoint; R ignores the reset with a notice) |

## 7. FAQ

**`Error: Please set the ROBOT_TYPE ...`** — export `ROBOT_TYPE` first; the
message lists every supported value.

**`uv sync` fails on a missing `limxsdk-*.whl`** — the `limxsdk-lowlevel`
submodule is not checked out. Run `git submodule update --init --recursive`.

**`limxsdk ... does not support the Centaur simulator side`** — the installed
SDK predates the Centaur API, so `DASF_*` cannot run. Update the
`limxsdk-lowlevel` submodule to a release that exposes `RobotType.Centaur` and
the LowerBody/UpperBody `*ForSim` methods.

**`Error: none of ['robot.xml'] exists under ...`** — the `robot-description`
submodule is missing, or that variant's assets are not published yet.

**`No module named limxsdk`** — the environment was created without the extra.
Run `uv sync --extra sdk`. Note that a later plain `uv sync` removes it again.
On a pip install, the wheel was not installed, or it was installed into a
different interpreter than the one running `simulator.py`.

**`python3 -m venv` fails with an `ensurepip` error** — on Debian and Ubuntu the
standard library venv module is packaged separately: `sudo apt install
python3-venv`. Or use option A, which needs no system package.

**The robot loads but never moves** — no controller is publishing, or (on
`DASF_*`) its commands lack `motor_names` and the native layer is rejecting
them.

**Simulator output disappears when piped** — fixed; if you see it on an older
checkout, set `PYTHONUNBUFFERED=1`.

## License and attribution

Apache License, Version 2.0. See [`LICENSE`](LICENSE); SPDX identifier
`Apache-2.0`.

- [`NOTICE`](NOTICE) — required attribution notice.
- [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md) — per-submodule and
  per-dependency provenance.
- [`SECURITY.md`](SECURITY.md) — how to report a vulnerability, and the
  simulation-versus-hardware boundary.
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — development workflow, submodule
  procedure, DCO sign-off.
- [`CHANGELOG.md`](CHANGELOG.md) — release notes and items blocked on upstream.

## Scope

**Included:** `simulator.py`, the `tron2_sim/` package (including the shipped
gripper calibration under `tron2_sim/config/`), and documentation. Submodules
are *declared* (pinned by commit) but not vendored:

- `robot-description/` — URDF / MJCF models and meshes.
- `robot-joystick/` — gamepad helper binary used to drive a controller
  by hand (see [§3a Gamepad control](#3a-gamepad-control)).
- `limxsdk-lowlevel/` — LimX low-level SDK with pre-built wheels.

**Excluded by design:** trained control policies (`.onnx`, `.pt`, `.pth`,
`.ckpt`), SDK binaries or wheels committed into this tree, calibration values,
firmware, bag captures, and hard-coded private network addresses. Command
examples use the placeholder `<robot-ip>`; the default endpoint is `127.0.0.1`.

> **Distribution.** The primary public distribution point for this
> repository is GitHub:
> <https://github.com/limxdynamics/tron2_mujoco_sim>. The internal
> LimX GitLab is a mirror; open issues, PRs, and security reports on
> the GitHub repository.

## Cite and support

If you use this simulator in academic or public work, please cite the
repository:

```
@misc{limx_tron2_mujoco_sim_2026,
  title  = {TRON2 MuJoCo simulator},
  author = {LimX Dynamics},
  year   = {2026},
  howpublished = {\url{https://github.com/limxdynamics/tron2_mujoco_sim}}
}
```

- **Bug reports / feature requests:**
  [GitHub Issues](https://github.com/limxdynamics/tron2_mujoco_sim/issues).
- **Questions / integration help:**
  [GitHub Discussions](https://github.com/limxdynamics/tron2_mujoco_sim/discussions).
- **Security reports:** email `contact@limxdynamics.com`; see
  [`SECURITY.md`](SECURITY.md) for the sim-vs-real boundary note.
- **Company / commercial contact:** <https://www.limxdynamics.com>.
