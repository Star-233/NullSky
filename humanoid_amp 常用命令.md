---
tags:
  - 索引
---
## 执行训练任务
```bash
python -m humanoid_amp.train --task <taskID> [--headless] [--num_envs <num>]
```
例如
```bash
python -m humanoid_amp.train --task Isaac-G1-AMP-Test-Direct-v0 --headless --num_envs 2048
```

## play 权重
```bash
python -m humanoid_amp.play --task <任务名称> --checkpoint <权重文件路径>
```


## 面板数据查看

```bash
python -m tensorboard.main --logdir logs/skrl/
```