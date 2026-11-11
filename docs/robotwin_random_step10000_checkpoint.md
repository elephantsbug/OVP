# RoboTwin random：10000 步 checkpoint

[返回项目首页](../README.md) · [clean 10k](../backups/step_010000_20261026/README.md) · [CSE 5k](cse_step5000_checkpoint.md)

这是 **RoboTwin random observer / ego / object split v2** 上完成 10000 次参数更新的模型，训练 run 为 `obj_video_random_train_lr2e6_v1`。它从 **RoboTwin clean 10k** 权重初始化，之后又作为 **CSE 5k** 的初始化来源。

random 和 clean 的文件都叫 `step_010000.pt`，但模型权重、数据划分和 SHA-256 不同，应分别下载到独立目录。

## 权重与版本

2026-11-11 通过 Hugging Face API 核对远端文件元数据：

| 项目 | 内容 |
| --- | --- |
| 模型仓库 | [elephantsbug/vpp2-object-video-random-step10000-20261102](https://huggingface.co/elephantsbug/vpp2-object-video-random-step10000-20261102) |
| 当前访问状态 | 公开；HF 原 README 中的 `Private` 是备份时描述 |
| 固定 revision | `b52fa4880298e7b3d856ca575eb4b3dc59289a2b` |
| 权重文件 | `step_010000.pt` |
| 文件大小 | 32,789,968,889 bytes，约 30.54 GiB |
| SHA-256 | `d4037927eaba7471c85ed9629ab4ab8120bf3f2cd1f1489cf7622427ac35c230` |
| 训练完成记录 | 2026-11-01 06:22:53（北京时间） |
| 备份范围 | DiT 权重、训练配置、checkpoint 元数据、README、SHA256SUMS |

远端 LFS 大小与 SHA-256 匹配 [checkpoint.json](https://huggingface.co/elephantsbug/vpp2-object-video-random-step10000-20261102/blob/b52fa4880298e7b3d856ca575eb4b3dc59289a2b/checkpoint.json)；此次整理未重新下载整份权重。HF 同时保留 [训练配置](https://huggingface.co/elephantsbug/vpp2-object-video-random-step10000-20261102/blob/b52fa4880298e7b3d856ca575eb4b3dc59289a2b/training_config.yaml) 和 [文件校验清单](https://huggingface.co/elephantsbug/vpp2-object-video-random-step10000-20261102/blob/b52fa4880298e7b3d856ca575eb4b3dc59289a2b/SHA256SUMS)。

## 输入、输出与数据

| 内容 | 设置 |
| --- | --- |
| 画面布局（宽 × 高） | observer 左 / ego 右，各 224×224，共 448×224 |
| RGB 条件 | episode 首帧 + 含当前帧的 8 帧历史窗口，共 9 帧 |
| 历史 stride | 30，按去重后的帧索引采样 |
| 物体未来 | 16 帧，stride 5 |
| 训练序列 | 9 帧 RGB + 16 帧黑背景物体视频，共 25 帧 |
| Prompt 缓存 | 冻结 UMT5，BF16 `[128,4096]`，真实 bool padding mask 与 token IDs |

v2 使用 2500 条 prompt 里可用的 2498 条，排除 2 条待复核记录；包含 2248 条训练 episode，另外 250 条 held-out episode 随后分为 **100 条验证、150 条最终测试**。各集合按完整 episode 隔离，覆盖既有任务，不是未见任务泛化测试。文本缓存包含 1161 种唯一 prompt，完整文本为 55–112 个有效 token，无截断。

数据与缓存分别位于原服务器本地 `data/robotwin_random_observer_ego_obj_splits_v2/` 和 `data/text_embeddings/robotwin_random_prompt_subset_v2/`。这些清单、媒体、去重 NPZ 和 embedding 未包含在此 HF 权重备份中。去重后的 stride 不能直接按 MP4 标称 FPS 换算固定秒数。

## 训练过程与参数

先从 clean 10k 权重进行学习率试验，选定 `2e-6` 的第 500 步完整状态，再续训至 random 阶段第 10000 步。正式续训依次在 2000 / 4000 / 6000 / 8000 / 10000 步保存并验证，阶段切换恢复同一优化器、调度器与采样进度；10000 步总 cosine 日程不重置。

| 项目 | 设置 |
| --- | --- |
| 训练模块 | 仅 Video DiT；VAE / CLIP 冻结，文本 embedding 预计算 |
| 数值与分布式 | BF16，4 卡，DeepSpeed ZeRO-1 |
| batch | 每卡 4，梯度累积 8，有效 global batch 128 |
| 学习率 | 峰值 `2e-6`，最低 `1e-7`；warmup 200，cosine，最小比例 0.05 |
| 总步数 / 保存间隔 | 10000 / 2000 次参数更新 |
| RGB 前缀 loss 权重 | 1.0 |
| train / infer shift | 3.0 / 3.0 |
| 原权重位置 | `runs/obj_video_random_train_lr2e6_v1/checkpoints/weights/step_010000.pt` |

以上运行过程和数据数量核对自 H200 的 `docs/robotwin_random_training_v2.md`、`docs/robotwin_random_formal_training_lr2e6_v1.md` 及 `runs/obj_video_random_train_lr2e6_v1_control/status.json`，这些原始运行记录目前仍是服务器本地产物。

定期验证使用固定的 100 条验证 episode、200 个窗口、seed 0、4 步去噪、shift 3，最终 150 条测试 episode 不参与这轮定期选择。这里仅说明验证流程，不将训练完成等同于最终测试效果；不同评测报告的共同前景 mask 可能不同，不能直接拼接其 MAE 数值。

## 下载与恢复

下载全部五个备份文件，以便直接运行校验清单：

```bash
hf download elephantsbug/vpp2-object-video-random-step10000-20261102 \
  step_010000.pt checkpoint.json training_config.yaml README.md SHA256SUMS \
  --revision b52fa4880298e7b3d856ca575eb4b3dc59289a2b \
  --local-dir weights/robotwin-random-step10000
(cd weights/robotwin-random-step10000 && sha256sum -c SHA256SUMS)
```

`training_config.yaml` 是 **8000→10000 步续训段的配置快照**。其中 `resume` 指向原 H200 的 8000 步完整状态，该状态不在此次备份中；不能将这个路径当作可下载的恢复材料。

推理或新一轮微调仍需兼容的 VPP2 / Wan 代码、冻结的 VAE / CLIP、相应 tokenizer 与文本 embedding，以及输入视频和清单。新 prompt 还需文本编码器生成 embedding。迁移时要按用途处理原配置中的绝对路径与 `resume` 设置。

元数据记录源码基线为 `26f93d5`，同时明确 `source_git_has_tracked_changes=true`。因此仅检出这个 GitHub 提交并下载权重，不保证恢复了 random 运行时的全部代码修改和本地资源。当前 GitHub 的 clean 指南可帮助理解通用入口，但不能直接替代 random 的数据约定和运行配置。

该备份**不包含优化器、调度器或 RNG 状态**。它可用于推理或后续训练的权重初始化，不能单独严格接续原训练。CSE 后训练使用此 random 权重，其配置和备份范围见 [CSE 5k](cse_step5000_checkpoint.md)。
