---
tags:
  - 索引
---

## 激活环境

列出 `conda` 环境
```bash
conda env list
```

激活 `isaaclab` 环境
```bash
conda activate env_isaaclab
```

## 列出所有任务
```bash
python scripts/environments/list_envs.py
```

## 执行任务
```bash
python <train脚本路径> --task <task名称> [参数]
```

常见的训练算法脚本
- **RSL-RL** :
	- 可能位置：`scripts/reinforcement_learning/rsl_rl/train.py`
- **RL-Games**:
	- 可能位置：`scripts/reinforcement_learning/rl_games/train.py`
- **SKRL**:
	- 可能位置：`scripts/reinforcement_learning/skrl/train.py`
- **Stable-Baselines3 (SB3)**:
	- 可能位置：`scripts/reinforcement_learning/sb3/train.py`

### A. 基础配置参数

- **`--task`**: (必填) 注册在 Gymnasium 注册表中的任务 ID（例如：`Isaac-Cartpole-v0`, `Isaac-Ant-Direct-v0`）。
- **`--num_envs`**: 启动的并行环境数量（向量化环境数量）。例如 `--num_envs 1024`。
- **`--device`**: 指定运行设备，可选 `cuda` (默认) 或 `cpu`。
- **`--seed`**: 随机种子，用于结果复现。
- **`--distributed`**: 是否开启分布式训练（多 GPU）。
### B. 渲染与可视化参数

- **`--headless`**: 开启无头模式（不显示 GUI）。训练时为了提升 FPS 建议开启。
- **`--visualizer`**: 指定可视化后端。可选：
    - `omniverse`: (默认) 完整的 Isaac Sim 视窗。
    - `newton`: 轻量化 OpenGL 可视化，适合快速迭代。
    - `rerun`: 支持远程查看和时间轴回溯。
    - 可以同时开启多个：`--visualizer omniverse newton`。
- **`--enable_cameras`**: 在无头模式下依然允许渲染传感器数据（用于视觉任务或视频录制）。
- **`--livestream {1, 2}`**: 开启流媒体传输。`1` 为 WebRTC 公网流，`2` 为局域网流。
### C. 训练与模型参数 (train.py / play.py)
- **`--checkpoint`**: 指定已训练好的模型文件路径（用于 `play.py` 回放或断点续训）。
- **`--max_iterations`**: 设置训练的最大迭代次数。
- **`--run_name`**: 为当前运行指定一个名称，用于生成日志文件夹。
- **`--agent`**: 指定使用的学习代理配置入口点。
- **`--video`**: 运行过程中自动录制并保存视频。
- **`--video_length`**: 每段录制视频的步数。
- **`--video_interval`**: 每隔多少步录制一段视频。

## 播放(play)
```bash
python <play脚本路径> --task <task名称> --checkpoint <权重文件> [参数] 
```
- **`--num_envs`**: 环境数量
    - 示例：`--num_envs 1`
- **`--visualizer`**: 选择不同的可视化方式。
    - `--visualizer omniverse`: 默认的高画质模式。
    - `--visualizer newton`: 极简模式，运行非常流畅，适合纯看动作。
- **`--video`**: 自动将回放过程录制成视频，保存到 log 文件夹下。
- **`--use_pretrained_checkpoint`**: 