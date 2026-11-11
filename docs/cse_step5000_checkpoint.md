# CSE 5000 步 checkpoint：单视角物体视频预测

[返回项目首页](../README.md) · [权重与版本](#权重与版本) · [评测结果](#3000-步与-5000-步的固定窗口评测) · [下载与恢复](#下载与恢复)

本文对应 CSE 单视角 640×480 模型。RoboTwin clean 双视角 448×224 的代码、配置和恢复步骤见 [clean 训练与推理](robotwin_clean_training.md)；两种输入尺寸和配置不能直接混用。

这份记录对应 CSE 仿真数据集训练至 **5000 次参数更新**的 Video DiT 权重。模型根据单个 observer 视角的全景 RGB 历史和英文动作计划，预测同一视角下、黑背景上的目标物体运动视频。它从 [RoboTwin random 10000 步 checkpoint](robotwin_random_step10000_checkpoint.md) 初始化，再在 CSE 上后训练。

## 权重与版本

| 项目 | 内容 |
| --- | --- |
| Hugging Face 模型仓库 | [elephantsbug/vpp2-obj-generation-cse-step5000](https://huggingface.co/elephantsbug/vpp2-obj-generation-cse-step5000)（公开，2026-11-11 核对） |
| GitHub 代码仓库 | [elephantsbug/vpp2-obj-generation](https://github.com/elephantsbug/vpp2-obj-generation) |
| 权重文件 | `step_005000.pt` |
| 文件大小 | 32,789,968,889 bytes，约 30.54 GiB |
| checkpoint 格式 | `wan21_video_v1`，包含 `format`、完整 `dit` state dict、`step=5000` |
| SHA-256 | `1180b147125895728b7426a7f547a5d97884746fcc6f2fb0d88dff6d61629c45` |
| Hugging Face revision | `fa714f0b560606dba82d76b099bfd0b0e88d246d` |
| 上传状态及配套文件 | **已上传并校验**（2026-11-10T14:25:01.524368+00:00）：远端 SHA-256 与 32,789,968,889 bytes 均匹配本地文件；本次 Hugging Face 仅上传 `step_005000.pt`，GitHub 仅更新本说明和 README 入口 |

原训练文件为 `runs/obj_video_cse_continue_5000_v1/checkpoints/weights/step_005000.pt`。该文件保存完整 Video DiT 参数，不是 LoRA。训练器保存此文件时传入 `optimizer=None`，因此它可用于推理或新一轮训练初始化，但不能单独恢复原训练的优化器、学习率调度器和随机数状态。

## 输入、输出与数据

视频保持原生 **640×480、30 fps、单 observer 视角**，不拼接 wrist 视角，不做空间裁剪或重采样。配置中的 `video_size` 按 `[height, width]` 写为 `[480, 640]`。

| 内容 | 设置 |
| --- | --- |
| 全景条件视频 | `observation.images.table_overview` |
| 物体监督视频 | `observation.images.mask_object_amodal` |
| 监督含义 | 当前 episode 的目标物体 RGB，背景为黑；不是场景内所有物体 |
| 条件帧 | 全景首帧 `I_0`，加 `I_(s-210), I_(s-180), …, I_s`，共 9 帧 |
| 历史间隔 | stride=30，即相邻历史帧间隔 1 秒；8 帧局部历史覆盖 7 秒 |
| 未来帧 | `O_(s+5), O_(s+10), …, O_(s+80)`，共 16 帧 |
| 未来间隔 | stride=5，即 1/6 秒；最远预测至当前时刻后约 2.67 秒 |
| 边界处理 | 不足的历史用全景首帧补齐；未来监督必须完整 |
| 文本 | 已生成的英文详细动作计划，使用缓存的文本 embedding |

`amodal` 监督移除了外部遮挡，不等于物体在所有姿态、所有位置都完整可见；自身遮挡和相机视野限制仍存在。RGB 与物体视频共享逐帧索引，确保同一时间、同一像素坐标下配对。

数据源为 `CSE_data/ces-pick-place-20261028`。700 条 episode 中，只使用 prompt 无待复核标记的 **631 条**，排除其余 **69 条**。按 7 个物体类别分层、固定随机种子 42 划分：**训练 568 条，测试 63 条**，每类 9 条测试 episode。同一 episode 的所有窗口保持在同一划分内。

## 模型与训练设置

| 项目 | 设置 |
| --- | --- |
| 模型系列 | VPP2 / Wan2.1-I2V-14B；“14B”为模型系列名称 |
| 实际可训练参数 | 约 **16.395B**，以训练器选入优化器的参数统计为准 |
| 初始化 | RoboTwin random 10000 步 checkpoint |
| 训练模块 | 仅 Video DiT |
| 冻结模块 | VAE、CLIP；文本 embedding 预计算，训练时不加载文本编码器 |
| 数值与分布式 | BF16，4 张 H200，DeepSpeed ZeRO-1 |
| batch | 每卡 1，梯度累积 32，有效 batch = `1 × 4 × 32 = 128` |
| 学习率 | 峰值 `2e-6`，warmup 200 次参数更新 |
| 学习率调度 | cosine，调度总长度 10000 步，最小比例 0.025 |
| 本次终点 | `stop_after_steps=5000`；没有将 cosine 压缩成 5000 步周期 |
| 其他 | Adam betas `(0.9, 0.999)`，weight decay 0.01，grad norm clip 1.0 |
| 保存 | 每 1000 步保存 checkpoint |
| train / infer shift | 3.0 / 3.0 |

CSE 首次训练从 random 权重建立新的优化器和调度器；后续 CSE 训练阶段通过完整 trainer state 接续。本次 3000→5000 步继续沿用既有调度。

训练时序仍为 **9 帧全景 RGB + 16 帧物体未来 = 25 帧**。VAE 编码后为 `[B, 16, 7, 60, 80]`，对应 8400 个视频 token。RGB 条件前缀的 loss 权重保持 **1.0**；全景前缀与物体未来均参与训练损失。推理沿用当前联合去噪流程，不对已知 RGB 前缀做逐步强制替换，最终导出后 16 帧物体预测。

## 3000 步与 5000 步的固定窗口评测

比较使用测试集的 **28 个固定窗口、7 类物体、seed=0/1/2**，每个 checkpoint 共 84 条预测。两组使用相同 RGB 历史、prompt 和初始噪声，统一 **8 步去噪、shift=3**。展示视频左侧为全景 RGB GT，其余面板为对应预测，物体 GT 用于计算指标及逐帧核对。

| 指标 | CSE 3000 步 | CSE 5000 步 |
| --- | ---: | ---: |
| 共同前景 MAE ↓ | 0.140783 | **0.135886** |
| 物体轮廓 IoU ↑ | 0.381388 | **0.408587** |
| MAE 优于静止基线的比例 ↑ | 100% | 100% |
| 人工确认严重变暗、消失或碎裂的预测数 ↓ | 7 / 84 | **3 / 84** |
| 至少一个 seed 出现上述异常的窗口数 ↓ | 6 / 28 | **2 / 28** |

MAE 在每个窗口共同的前景并集上计算，前景并集覆盖 GT、静止基线以及两个 checkpoint 的三个 seed，RGB 误差除以 255。IoU 的前景阈值为 `max(R,G,B)>16`，在 16 帧上聚合交并区域。静止基线重复当前时刻的物体 GT，评测才读取该目标视频；模型输入不含物体 GT。汇总先平均同一窗口的 seed，再平均同类别窗口，最后等权平均物体类别。

5000 步在这批窗口上平均误差更低、轮廓重合更好，严重可见性异常更少。人工检查中仍可见运动幅度不足、过早停止、终点偏移和 seed 敏感性。例如 `red_cup/episode_0000599`、anchor=230 的 seed=0，5000 步预测在未来第 3–6 帧严重变暗，而 3000 步保持可见；不能据平均值认定每个案例都改善。

| CSE 5000 步 seed | 前景 MAE ↓ | IoU ↑ | 人工可见性异常 |
| --- | ---: | ---: | ---: |
| 0 | 0.139676 | 0.380359 | 2 / 28 |
| 1 | **0.130161** | **0.425722** | **0 / 28** |
| 2 | 0.137822 | 0.419680 | 1 / 28 |

seed=1 在本次窗口集合上最好，可用于这批案例的展示；正式比较仍应保留三个 seed，不能把事后挑选的最好 seed 当作整体能力。

这是一组 **28 个选定窗口的探索性评测**，不是整个测试集准确率，也不是机器人任务成功率。以窗口为采样单位、先平均 seed 的配对 bootstrap，MAE 改善的 95% 区间为 `[-0.00722, 0.01752]`，IoU 改善区间为 `[-0.01097, 0.06527]`，均跨过 0。因此可以报告“当前固定窗口的平均表现改善”，不能据此宣称总体提升已得到统计确认。

原始指标与人工检查记录来自 `runs/cse_3000_5000_multiseed_v1/review/metrics.json` 和 `visual_annotations.json`。本地 `runs/` 路径不意味着这些产物已随 GitHub 仓库上传；远端备份范围以本页顶部的文件清单为准。

## 下载与恢复

2026-11-11 通过 Hugging Face API 核对，仓库已公开，文件大小和 SHA-256 仍与下述固定 revision 一致；可直接下载。

```bash
hf download elephantsbug/vpp2-obj-generation-cse-step5000 step_005000.pt \
  --revision fa714f0b560606dba82d76b099bfd0b0e88d246d \
  --local-dir weights/cse-step5000
sha256sum weights/cse-step5000/step_005000.pt
```

将输出与本页 SHA-256 对照。命令固定到已核验的 revision，避免后续仓库更新导致下载内容变化。

推理还需要与该模型兼容的代码、CSE 的 `[480, 640]` 配置、冻结的 Wan VAE/CLIP，以及对应 prompt 的缓存 embedding。对于新 prompt，还需要 T5 文本编码器和 tokenizer 生成 embedding。权重文件本身不包含这些组件，也不包含训练视频、测试视频或原始清单。

冻结组件沿用项目现有 Wan2.1-I2V-14B-480P 依赖。已有备份记录的上游来源为 [`Haodong082399/VPP2`](https://huggingface.co/Haodong082399/VPP2)，revision `04824ddb3aae11f13db50984ed7d540c3b77dd9f`，其中 `checkpoints/Wan2.1-I2V-14B-480P/` 提供 VAE、CLIP、T5 等文件。上游可下载不等于它们已包含在此次备份中。

2026-11-10 的 checkpoint 备份没有将 CSE 源码、运行配置、环境文件或评测产物上传到远端；GitHub 更新范围是本说明及 README 入口。现有服务器上的 `src/vpp2_obj/`、`third_party/vpp2/` 和 CSE 配置仍是恢复依赖，不能将本次权重备份视为完整运行环境备份。CSE 版本放宽了原先固定双视角尺寸的配置校验，并让训练 probe 使用配置分辨率；仅恢复 GitHub 旧版 clean 代码和新权重不足以复现本次单视角运行。配置内原服务器的绝对路径也需要按新机器位置调整。

在保留当前 CSE 代码的环境中，复现上述评测时，必须显式指定兼容的 `--config`（分辨率 `[480, 640]`）、`--checkpoint` 和 `--steps 8`；不要沿用默认的双视角配置。`shift=3` 来自配置，命令行没有 `--shift` 参数。正式复现评测需分别使用 `--seed 0`、`--seed 1`、`--seed 2`。

需要严格接续第 5000 步训练时，必须另行保存并恢复完整 DeepSpeed trainer state。仅加载本页的 `step_005000.pt` 应视为“从现有权重开始新训练”，需要重新决定优化器和学习率调度，而不能声称恢复了原训练状态。
