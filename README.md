# MotrixArena S1 · 越障导航（Section01）

基于谋先飞官方仿真框架 [MotrixLab](https://github.com/Motphys/MotrixLab)（`MotrixArena-S1` 分支）实现的 **VBot 四足机器人越障导航** 任务。机器人从 Section01 起点出发，依次穿越起步平台、起伏崎岖区、出口落差坎与 15° 上坡，最终稳定站上 **2026 平台**，完成越障导航赛段第一阶段。

---

## 任务概览

第一阶段路线沿前进方向（Y 轴）分为五段：

| 路段（Y 范围，m） | 地形特征 | 主要难点 |
| --- | --- | --- |
| 起步平台（Y ≤ −1.5） | 平地，出生区 | 起步姿态稳定 |
| 起伏崎岖区（−1.5 ~ 1.5） | 地面起伏约 0.28 m | 抬腿不够高会被绊倒 |
| 落差坎（Y ≈ 1.5） | 最高约 0.28 m 后骤降至平地 | 前腿先下、后腿没跟上易栽倒 |
| 上坡段（2.0 ~ 6.9） | 真实 15° 斜坡 | 冲折角栽头、后腿缺爬坡动力 |
| 2026 平台（Y ≥ 6.9） | 平台，高约 0.66 m | 上台后站稳、不掉落 |

---

## 环境要求

- Python 3.10
- [UV](https://docs.astral.sh/uv/) 包管理工具
- Linux（推荐 Ubuntu）
- 训练框架：SKRL（PyTorch 后端），PPO 算法

## 安装与部署

### 1. 克隆官方仓库（指定分支）

```bash
git clone --branch MotrixArena-S1 https://github.com/Motphys/MotrixLab.git
cd MotrixLab
git lfs pull
uv sync --all-packages --extra skrl-torch
```

### 2. 接入本项目的 navigation 包

将本仓库提供的文件按以下方式放入 MotrixLab：

1. 把 `vbot_navigation` 包解压/复制到 `motrix_envs/src/motrix_envs/` 下，得到 `navigation` 文件夹（其中包含 `vbot` 子目录）。
2. 删除 `motrix_envs/src/motrix_envs/locomotion/` 下的 `anymal_c` 文件夹，并把该目录的 `__init__.py` 改为：

   ```python
   from . import go1, go2
   ```
3. 修改 `motrix_envs/src/motrix_envs/__init__.py`，加入 `navigation`：

   ```python
   from . import basic, locomotion, manipulation, navigation  # noqa: F401
   ```
4. 把本仓库的 `cfgs.py` 复制到 `motrix_rl/src/motrix_rl/` 下，替换原有同名配置文件。

### 3. 放置地形资产

官方初始资产的越障导航地形不完整（红包等可视化缺失、三段地形未拼接、2026 平台处会掉落）。将更新版地形资产（XML 场景文件与对应网格）放入项目的 `xmls/` 目录后，`scene_section01.xml` 即可正确加载 Section01 地形，无需改动路径。

---

## 运行

所有命令均在 `uv` 管理的环境下运行，通过环境名 `vbot_navigation_section01` 指定任务。

```bash
# 可视化查看环境（确认地形与机器人加载正常）
uv run scripts/view.py --env vbot_navigation_section01

# 训练（4096 并行环境）
uv run scripts/train.py --env vbot_navigation_section01 --num-envs 4096

# 训练过程可视化
uv run tensorboard --logdir runs/vbot_navigation_section01

# 加载权重做推理 / 录屏（单环境）
uv run scripts/play.py --env vbot_navigation_section01 --num-envs 1
```

训练结果（权重、日志）默认保存在 `runs/vbot_navigation_section01/`。

---

## 方法概述

### 分段路径点引导

沿路线在前进方向布置 7 个路径点作为「途经门」，机器人到达即切换下一个，缓解长距离稀疏奖励的探索困难：

```
(0,-0.60) → (0,1.20) → (0,2.25) → (0,4.00) → (0,6.00) → (0,7.00) → (0,7.80)
```

指令在机身坐标系下由 P 控制器（线速度系数 0.8、角速度系数 1.0）生成，经速度限幅与一阶低通平滑（τ = 0.25 s）输出；按路段分级限速，崎岖/落差/上坡段降速更稳。

### 前方地形感知观测（68 维）

在机器人前方 8 个固定距离 `[0.20 ~ 1.60] m` 处采样归一化地形高度，让策略提前感知落差与坡度。崎岖区只标记「此处粗糙」而不伪造起伏形状，避免误导。观测构成：基础状态 48 维 + 足端接触力 12 维 + 前方地形采样 8 维。

### 越障难点分段塑形奖励（自定义）

针对每个失败现象设计「分区生效」的塑形奖励，是本项目的核心：

- `per_leg_swing` / `drop_leg_catchup`：逼每条腿（尤其后腿）迈步、在落差点跟上；
- `swing_foot_height`：崎岖区抬腿跨越起伏；
- `slope_leg_drive` / `slope_front_drive` / `slope_pitch` / `slope_hip`：上坡段持续蹬地、顺坡微前倾、约束髋关节外张防栽头。

### 控制与终止

PD 关节位置控制（`action_scale = 0.5`，KP = 80，KD = 6，力矩限幅 ±17/±34 N·m），仿真 100 Hz，单回合 40 s / 4000 步；以基座触地为主终止，数值异常做兜底。

---

## 结果

训练后机器人可从 Section01 起点稳定通过崎岖区、落差坎与上坡，最终站上 2026 平台不掉落，达成越障导航赛段第一阶段完成标准。到达目标成功率随训练推进上升并趋于稳定。

![越障导航演示](demo.gif)

---

## 目录结构

```
.
├── README.md
├── vbot/                     # navigation 环境包（放入 motrix_envs/src/motrix_envs/navigation/）
│   ├── cfg.py                # 环境配置（注册名 vbot_navigation_section01）
│   ├── vbot_section01_np.py  # Section01 越障导航环境实现
│   └── xmls/                 # 地形/场景资产
├── runs/
│   └── vbot_navigation_section01/   # 训练权重与日志（用于 play.py 加载测试）
└── demo.gif                  # 演示动图
```

> 注 1：以上为本项目提供的文件，需配合官方 MotrixLab（`MotrixArena-S1` 分支）使用，详见「安装与部署」。
>
> 注 2：训练超参 `cfgs.py` 是放在 `motrix_rl/` 下的，不在 `vbot/` 里；本仓库未单独附带，复现训练时请参照「安装与部署」放置。

---

## 致谢与许可

- 仿真框架与基础环境来自 [Motphys/MotrixLab](https://github.com/Motphys/MotrixLab)（Apache-2.0）。
- 本项目为 MotrixArena S1 结营作业实践，沿用上游 Apache-2.0 许可证。

