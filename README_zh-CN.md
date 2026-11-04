<!--
  SPDX-FileCopyrightText: 2024-2026 LimX Dynamics Technology Co., Ltd.
  SPDX-License-Identifier: Apache-2.0
-->

<p align="center">
  <img src="doc/banner.png" alt="TRON2 MuJoCo 仿真器：由 robot-description 渲染的 DA、SF、DASF、WF、DACH 五种 TRON2A 模型" width="100%">
</p>

<p align="center">
  <a href="README.md">English</a> | <b>中文</b>
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-Apache--2.0-2f6feb" alt="License: Apache-2.0"></a>
  <img src="https://img.shields.io/badge/python-%E2%89%A53.10-2f6feb" alt="Python >= 3.10">
  <img src="https://img.shields.io/badge/MuJoCo-%E2%89%A53.2.2-2f6feb" alt="MuJoCo >= 3.2.2">
  <img src="https://img.shields.io/badge/limxsdk-4.0.2-2f6feb" alt="limxsdk 4.0.2">
</p>

# tron2-mujoco-sim

> **TRON2 开源生态的一部分。** 上手路线和其他 TRON2 仓库：[TRON2 快速上手](https://github.com/limxdynamics/limxdynamics/blob/main/doc/quickstart_cn.md#tron2)。

TRON2 系列机器人的 MuJoCo 仿真器。它把 LimX 底层 SDK（`RobotCmd` / `RobotState`
承载 `q` / `dq` / `tau` / `Kp` / `Kd`，以及 `ImuData` 与夹爪消息）桥接到
`mujoco.MjData`，使得驱动真机的同一套控制器线格式可以直接对着仿真跑。

构型是声明出来的，不是写死的：一个构型 = 若干关节通道 + 可选模块，在
`tron2_sim/variants/` 里拼装。所有受支持构型的所有通道都走 SDK 的 `*ForSim`
仿真端接口——不存在第二套消息栈。

> **适用范围：** 本仓库只是仿真端，自身不运行任何策略。请通过 `limxsdk` 接入你的
> 控制器，例如 [tron2_rl_deploy_python](https://github.com/limxdynamics/tron2_rl_deploy_python)。
> 仿真器是硬件接口的镜像，不是安全兜底（见 [`SECURITY.md`](SECURITY.md)）。

## 亮点

- **与真机相同的线格式。** 控制器通过 `limxsdk` 发送 `RobotCmd`、读取
  `RobotState` / `ImuData`，与在真机上完全一样，只是换了端点。
- **一个入口，十种构型。** SF、WF、DA、DACH、DASF，各有 TRON2A 与 TRON2B 两版，
  用 `ROBOT_TYPE` 选择。
- **按关节名映射。** 线序按关节名解析，即使模型 XML 的执行器声明顺序不同，也能与真机对齐。
- **稳定的 1 kHz 物理。** 物理步进与状态发布在独立线程上运行；viewer 用另一份
  `MjData` 渲染，不会拖慢它们。
- **支持无图形运行。** `--headless --duration 30` 在 CI 或远程机器上运行同样的物理与通信。
- **自带 DACH 2F 夹爪。** 连杆夹爪模块，标定参数位于 `tron2_sim/config/gripper_config.yaml`。

## 型号一览

| | 型号 | 说明 | 自由度 | 格式 |
|:---:|---|---|:---:|---|
| <img src="doc/model_sf.png" alt="SF_TRON2A" width="320"> | **SF**<br>`SF_TRON2A`<br>`SF_TRON2B` | 掌足双足。一个 SDK 通道：`/motor/cmd\|state` + `/ImuData`。 | 10 | MJCF · URDF · USD |
| <img src="doc/model_wf.png" alt="WF_TRON2A" width="320"> | **WF**<br>`WF_TRON2A`<br>`WF_TRON2B` | 轮足双足：8 个腿部关节和 2 个轮子在同一通道上。 | 10 | MJCF · URDF · USD |
| <img src="doc/model_da.png" alt="DA_TRON2A" width="320"> | **DA**<br>`DA_TRON2A`<br>`DA_TRON2B` | 支架上的双臂，每侧 7 个关节。 | 14 | MJCF · URDF · USD |
| <img src="doc/model_dach.png" alt="DACH_TRON2A" width="320"> | **DACH**<br>`DACH_TRON2A`<br>`DACH_TRON2B` | 双臂 + 2 自由度头。默认加载带 2F 夹爪的 `robot_grasper.xml`；`--no-grasper` 改用 `robot.xml`。 | 16 | MJCF · URDF · USD |
| <img src="doc/model_dasf.png" alt="DASF_TRON2A" width="320"> | **DASF**<br>`DASF_TRON2A`<br>`DASF_TRON2B` | 拼人形：SF 双足 + DACH 双臂和头，分两个通道（下身 10、上身 16）。 | 26 | MJCF · URDF · USD |

模型来自 [`robot-description`](https://github.com/limxdynamics/tron2-robot-description)
submodule；仿真器加载其中的 MJCF，TRON2A 另外提供 Xacro 源文件。
图片由 MuJoCo 以同一套相机和灯光渲染；DASF 的肩部向下转到自然下垂，其余为零位姿态。

## 演示

均由本仿真器录制，每个构型一段。

| `SF_TRON2A` | `WF_TRON2A` |
|:---:|:---:|
| <img src="doc/gif/SF_TRON2A.gif" alt="SF_TRON2A 演示" width="100%"> | <img src="doc/gif/WF_TRON2A.gif" alt="WF_TRON2A 演示" width="100%"> |
| **`DACH_TRON2A`** | **`DASF_TRON2A`** |
| <img src="doc/gif/DACH_TRON2A.gif" alt="DACH_TRON2A 演示" width="100%"> | <img src="doc/gif/DASF_TRON2A.gif" alt="DASF_TRON2A 演示" width="100%"> |

媒体规范见 [`doc/gif/README.md`](doc/gif/README.md)，由 CI 强制。

## 工作原理

<p align="center">
  <img src="doc/architecture_zh.png" alt="控制器经 limxsdk、SdkBus、JointChannel 进入 1 kHz 物理线程，状态与 IMU 原路发回；渲染线程单独驱动 viewer" width="100%">
</p>

`ROBOT_TYPE` 从 `tron2_sim/variants/` 选出构型，构型声明了自己的通道和模块。
每个物理周期对最新命令应用 MIT 控制律，让 MuJoCo 步进 1 ms，读取关节与 IMU
状态，再经 SDK 发布回去。

## 快速开始

不需要 ROS，也不需要控制器，就能在 viewer 里加载机器人（需要
[uv](https://docs.astral.sh/uv/)；`robot-description` 检出后约 1.7 GB）：

```bash
git clone https://github.com/limxdynamics/tron2_mujoco_sim.git
cd tron2_mujoco_sim
git submodule update --init --depth 1 robot-description limxsdk-lowlevel
uv sync --extra sdk
ROBOT_TYPE=DASF_TRON2A uv run simulator.py
```

在控制器连上 `127.0.0.1` 之前，关节没有力矩，带腿的模型会瘫倒在地；按**空格**暂停后
可以用滑条手动摆姿态。启动
[tron2_rl_deploy_python](https://github.com/limxdynamics/tron2_rl_deploy_python)
等控制器即可驱动它。详细步骤和 pip 方式见[依赖与部署](#1-依赖与部署)。

## 1. 依赖与部署

| 依赖 | 版本 | 说明 |
|---|---|---|
| Python | >= 3.10 | 已在 3.10 验证 |
| `mujoco` | >= 3.2.2 | 物理与 passive viewer |
| `PyYAML` | >= 6.0 | 读 `tron2_sim/config/gripper_config.yaml` |
| `limxsdk` | 4.0.2 | arch 相关的 wheel，位于 `limxsdk-lowlevel` submodule 内；单独安装且必须带 `--no-deps` |

两种方式都先克隆：

```bash
git clone --recurse-submodules https://github.com/limxdynamics/tron2_mujoco_sim.git
cd tron2_mujoco_sim
```

若此前未带 submodule 克隆，先执行 `git submodule update --init --recursive`。

### 方式 A：uv（推荐）

[uv](https://docs.astral.sh/uv/) 会自动装好对应版本的 Python，按仓内已提交的
`uv.lock` 解析全部依赖，无需手动管理虚拟环境：

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh   # 安装 uv
uv sync --extra sdk                               # 物理 + SDK

export ROBOT_TYPE=SF_TRON2A
uv run simulator.py                               # 图形模式
uv run simulator.py --headless                    # 无图形（CI / 远程）
uv run simulator.py --headless --duration 30      # 运行 30 秒后退出

uv run simulator.py --headless --duration 30 --no-grasper  # 不加载 DACH 2F 夹爪
```

`uv sync` 需要 `limxsdk-lowlevel` submodule 已检出——SDK wheel 就在里面。之后若
再执行一次不带 `--extra sdk` 的 `uv sync`，`limxsdk` 会被移除。

### 方式 B：pip

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -U pip
pip install "mujoco>=3.2.2" "pyyaml>=6.0"

# 按机器架构选 SDK wheel。--no-deps 是必须的：该 wheel 自带的元数据会拖进
# onnxruntime/pygame/scipy/pandas，并把 numpy 钉在 <1.26.4。
pip install --no-deps limxsdk-lowlevel/python3/amd64/limxsdk-*.whl     # x86_64
# pip install --no-deps limxsdk-lowlevel/python3/aarch64/limxsdk-*.whl # aarch64

export ROBOT_TYPE=SF_TRON2A
python3 simulator.py                              # 图形模式
python3 simulator.py --headless --duration 30
```

### 环境变量

| 变量 | 默认值 | 作用 |
|---|---|---|
| `ROBOT_TYPE` | *（必填）* | 选择构型，如 `SF_TRON2A` |
| `ROBOT_IP` | `127.0.0.1` | SDK 端点 |
| `GRIPPER_CONFIG` | `tron2_sim/config/gripper_config.yaml` | 改用其它 DACH 夹爪标定文件 |
| `DACH_GRASPER` | `1` | 置 `0` 改加载 `robot.xml` 而非 `robot_grasper.xml`，等价于 `--no-grasper` |

## 2. 支持的构型

共 10 型：5 个基名 × `TRON2A` / `TRON2B` 两族。两族的关节名与线序完全一致，
控制器代码无需改动即可分别连接。

| 基名 | TRON2A | TRON2B | 说明 |
|---|---|---|---|
| SF | `SF_TRON2A` | `SF_TRON2B` | 掌足双足（10 关节）|
| WF | `WF_TRON2A` | `WF_TRON2B` | 轮足双足（10 关节）|
| DA | `DA_TRON2A` | `DA_TRON2B` | 双臂（14 关节）|
| DACH | `DACH_TRON2A` | `DACH_TRON2B` | 双臂 + 2 自由度头（16 关节），可选 2F 夹爪 |
| DASF | `DASF_TRON2A` | `DASF_TRON2B` | 拼人形：SF 双足 + DACH 双臂头（26 关节）|

## 3. 键盘控制（图形模式）

- **空格** —— 暂停 / 恢复。暂停即手动模式：右侧 Control 滑条直接摆位关节。
  （DACH 2F 夹爪在手动模式空转，既不跟滑条也不跟命令。）
- **R** —— 仅复位浮动基座位姿，关节角保留。固定基座构型打印提示后忽略。
- **Backspace / Reset 按钮** —— 全量复位（选中 keyframe 时按 keyframe）。
- **双击选中 + Ctrl 拖拽** —— 施加扰动力。

### 3a. 手柄操控

对于运行姊妹仓库 [`tron2_rl_deploy_python`](https://github.com/limxdynamics/tron2_rl_deploy_python)
部署栈的 SF/WF 构型手柄操控流程，
先初始化 `robot-joystick` submodule，再与控制器一起运行辅助程序：

```bash
./robot-joystick/robot-joystick
```

默认按键映射：

- `L1 + Y` —— 切换到 WALK。
- `L1 + X` —— 切回 IDLE。
- `R1` —— 清空速度指令。

## 4. 通信

<p align="center">
  <img src="doc/channels_zh.png" alt="SF、WF、DA、DACH、DASF 的通道、SDK 话题与线序" width="100%">
</p>

**线序契约**（cmd/state 下标序，2A/2B 相同）：DACH 头为 pitch→yaw，DASF 头为
**yaw→pitch**。个别模型的 XML 执行器声明序与线序不同，由仿真按关节名解析吸收
——绝不按位置假设。

行为要点：

- **Centaur 通道（DASF）的 cmd 建议携带关节名**（`RobotCmd.motor_names`）。native
  层会拿它与已发布的 state 做校验，不匹配则拦截并刷
  `ERROR: Centaur ... RobotCmd does not match the corresponding RobotState`。
  仿真端发布的 state 始终带真实关节名。
- DACH 2F 夹爪标定见 `tron2_sim/config/gripper_config.yaml`。
- DACH grasper 模型上，`grasper_base_{L,R}_Joint_ctrl` 由连杆机构带动而非由通道
  驱动，其 `ctrl` 恒为 0，启动时会打印一行 `INFO: unowned actuators`，属预期。

## 5. 架构

```
simulator.py            入口：ROBOT_TYPE → 构型注册表 → SimCore
tron2_sim/
  spec.py               线序 + 构型规格（纯数据）
  core.py               物理/渲染双线程、双 MjData 快照、手动模式、复位
  channels.py           JointChannel：按名 joint_map + MIT 控律 + state/IMU 出站
  transports.py         SdkBus（limxsdk *ForSim）与能力守卫
  config/               YAML 加载 + 随仓发布的 gripper_config.yaml
  modules/              可选能力；dach_grasper（2F 连杆夹爪）
  variants/             每基名一个文件；__init__.py 是注册表
doc/gif/                演示动图（见上方演示）
```

线程模型：物理线程独占物理 `MjData`，渲染线程独占 `render_data`；UI 复位与拖拽力
经标志与缓冲移交，`MjData` 永远单写者。

## 6. TRON2B 与 TRON2A 的差异

<p align="center">
  <img src="doc/tron2a_vs_tron2b_zh.png" alt="SF_TRON2A 与 SF_TRON2B 并排对比，标注膝关节限位、力矩与基座质量" width="100%">
</p>

关节命名、顺序与话题逐字节相同，控制器代码无需改动。但 2B 是硬件换代，
**为 2A 训练/整定的策略与增益不可直接套用**。

| 差异项 | TRON2A | TRON2B |
|---|---|---|
| 髋 pitch/roll、膝力矩 | ±150 N·m | **±200 N·m**（armature 同步更新）|
| 髋 yaw、踝 pitch、肘力矩 | ±60 / ±70 N·m | **±70 N·m** |
| 腕、头力矩 | ±20 N·m | **±15 N·m** |
| `base_Link` 质量（SF/WF/DA/DACH）| 12.57 kg | **13.4 kg**（质心/惯量同步更新）|
| SF 膝 / 髋 yaw 限位 | `knee: [-2.618, 0.262]` | **符号翻转** `knee: [-0.262, 2.618]`，连杆几何随之镜像 |
| DA / DACH 基座 | 浮动（近刚性 freejoint）| **固定**（无 freejoint，R 键打印提示后忽略）|

## 7. FAQ

**`Error: Please set the ROBOT_TYPE ...`** —— 先 export `ROBOT_TYPE`，报错信息里
列出了全部合法值。

**`uv sync` 报缺 `limxsdk-*.whl`** —— `limxsdk-lowlevel` submodule 未检出，
执行 `git submodule update --init --recursive`。

**`limxsdk ... does not support the Centaur simulator side`** —— 当前 SDK 版本
早于 Centaur 接口，`DASF_*` 跑不了。需把 `limxsdk-lowlevel` submodule 更新到带
`RobotType.Centaur` 与 LowerBody/UpperBody `*ForSim` 方法的版本。

**`Error: none of ['robot.xml'] exists under ...`** —— `robot-description`
submodule 缺失，或该构型的资产尚未发布。

**`No module named limxsdk`** —— 建环境时没带 extra，执行 `uv sync --extra sdk`。
注意之后再跑一次不带 extra 的 `uv sync` 会把它移除。若走 pip 安装，则是 wheel 没装，
或装到了另一个解释器里（不是跑 `simulator.py` 的那个）。

**`python3 -m venv` 报 `ensurepip` 错误** —— Debian / Ubuntu 上标准库的 venv 模块
是单独打包的：`sudo apt install python3-venv`。或者改用方式 A，它不需要任何系统包。

**机器人加载了但一动不动** —— 没有控制器在发命令；或（`DASF_*` 的情形）命令未带
`motor_names`，被 native 层拦截。

**输出重定向到管道后消失** —— 已修复；旧检出上遇到时可设 `PYTHONUNBUFFERED=1`。

## 许可与归属

Apache License 2.0，见 [`LICENSE`](LICENSE)，SPDX 标识 `Apache-2.0`。

- [`NOTICE`](NOTICE) —— 归属声明
- [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md) —— 逐 submodule、逐依赖的来源与许可
- [`SECURITY.md`](SECURITY.md) —— 漏洞报告流程，以及仿真与真机的边界
- [`CONTRIBUTING.md`](CONTRIBUTING.md) —— 开发流程、submodule 更钉步骤、DCO 签署
- [`CHANGELOG.md`](CHANGELOG.md) —— 版本说明与受上游阻塞项

## 收录范围

**收录：** `simulator.py`、`tron2_sim/` 包（含随仓发布的夹爪标定，位于
`tron2_sim/config/`）与文档。submodule 只按 commit **声明**，不入仓：

- `robot-description/` —— URDF / MJCF 模型与网格
- `robot-joystick/` —— 手柄辅助程序，用于手动操控控制器（见
  [§3a 手柄操控](#3a-手柄操控)）
- `limxsdk-lowlevel/` —— LimX 底层 SDK 及预编译 wheel

**刻意不收录：** 训练好的控制策略（`.onnx`、`.pt`、`.pth`、`.ckpt`）、直接提交进
本仓的 SDK 二进制或 wheel、标定值、固件、bag 录制，以及任何硬编码的私网地址。
命令示例一律用占位符 `<robot-ip>`，默认端点是 `127.0.0.1`。

> **发布渠道。** 本仓库的主发布点为 GitHub：
> <https://github.com/limxdynamics/tron2_mujoco_sim>。
> LimX 内部 GitLab 为镜像；Issue、PR 与安全报告请提交到 GitHub。

## 引用与支持

如果你在学术或公开工作中使用了本仿真器，请引用本仓库：

```
@misc{limx_tron2_mujoco_sim_2026,
  title  = {TRON2 MuJoCo simulator},
  author = {LimX Dynamics},
  year   = {2026},
  howpublished = {\url{https://github.com/limxdynamics/tron2_mujoco_sim}}
}
```

- **Bug 反馈 / 功能建议：**
  [GitHub Issues](https://github.com/limxdynamics/tron2_mujoco_sim/issues)。
- **使用问题 / 集成帮助：**
  [GitHub Discussions](https://github.com/limxdynamics/tron2_mujoco_sim/discussions)。
- **安全问题上报：** 邮箱 `contact@limxdynamics.com`；仿真 vs 实机
  的边界说明见 [`SECURITY.md`](SECURITY.md)。
- **公司 / 商务联系：** <https://www.limxdynamics.com>。
