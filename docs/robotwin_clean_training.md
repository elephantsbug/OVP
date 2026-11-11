# RoboTwin clean：训练与推理

[返回项目首页](../README.md) · [数据与 prompt](robotwin_clean_data.md) · [权重与恢复](../backups/step_010000_20261026/README.md)

本文对应 **RoboTwin clean、observer / ego 双视角、448×224** 版本。训练已完成 10000 次参数更新；归档日志记录于 2026-10-23 保存最终权重和 trainer state。CES 的单视角 640×480 模型请使用 [CES 专页](cse_step5000_checkpoint.md)。

## 实际训练配置

以 [已保存的运行配置](../backups/step_010000_20261026/metadata/runs/obj_video_train_v1/config.yaml) 和 [训练日志](../backups/step_010000_20261026/metadata/runs/obj_video_train_v1/train.log) 为准。可编辑配置入口为 [obj_video_train_v1.yaml](../configs/obj_video_train_v1.yaml)。

| 项目 | clean 10k 的实际设置 |
| --- | --- |
| 初始化 | VPP2 Stage-2 `vpp2-video-stage2-17f.pth` |
| 训练模块 | Video DiT；VAE / CLIP 冻结，文本使用预计算 UMT5 embedding |
| 分布式与精度 | 4 卡，DeepSpeed ZeRO-1，BF16 |
| 每卡 batch / 梯度累积 | 4 / 8 |
| 有效 global batch | `4 × 4 × 8 = 128` |
| 学习率 | `2e-6`，warmup 200，cosine，最小比例 0.025 |
| 总步数 / 保存间隔 | 10000 / 2000 次参数更新 |
| 条件与输出 | 9 帧全景 RGB + 16 帧物体未来 |
| 训练视频形状 | `[B, 3, 25, 224, 448]` |
| 历史 / 未来 stride | 30 / 5，均在去重后的索引空间采样 |
| RGB 前缀 loss 权重 | 1.0，保留上游总 loss 归一化 |
| train / infer shift | 3.0 / 3.0 |

**不要将早期 baseline 的 batch 832、学习率 `5e-6` 当作本次训练参数。** `reference_global_batch_size: 832` 仍保留在继承配置中，但实际 batch 由每卡 batch、卡数和累积次数计算。对应的早期设计、DeepSpeed forward 修复和局部验证记录见 [基线设计文档](obj_video_baseline_v1.md)。

模型联合去噪全部 7 个 latent 时间位置，解码 25 帧后导出后 16 帧物体预测。推理不逐步强制替换 RGB 前缀，也不将未来物体 GT 作为输入。

## 使用前准备

1. 按 [恢复指南](../backups/step_010000_20261026/README.md) 解包 `third_party/vpp2/`，恢复训练环境并下载 clean 10k 权重及冻结组件。
2. 准备 [数据指南](robotwin_clean_data.md) 中的 RGB 视频、去重索引、train / test 清单、tokenizer 与文本 embedding。仓库内的清单快照不包含它们引用的媒体。
3. 检查 `env_train.sh`、配置和清单中的绝对路径。迁移到新机器时需一致地更新路径、哈希与缓存约定，不能只设置 `VPP2_PROJECT_ROOT`。

以下命令在项目根目录、依赖与本地数据已恢复后使用。

## 推理

先用测试集里的一个窗口检查配置与索引。`--anchor` 是去重后的帧序号；示例权重路径与恢复指南中的下载目录一致。

```bash
source env_train.sh
python -m vpp2_obj.infer \
  --config configs/obj_video_train_v1.yaml \
  --checkpoint weights/robotwin-clean-step10000/step_010000.pt \
  --task adjust_bottle --episode episode_0000016 --anchor 98 \
  --output runs/clean_step10000_infer_new --dry-run
```

`--dry-run` 在 CPU 上检查配置、清单和索引，不解码视频、不写输出。确认后移除 `--dry-run`，通过 `CUDA_VISIBLE_DEVICES` 选择空闲 GPU，并使用一个不存在的输出目录。推理生成 `future.mp4` 和记录输入索引、prompt、checkpoint 来源的 `trace.json`。

`--steps` 可显式指定去噪步数，默认取配置的 `eval_num_inference_steps`（该配置为 10）。`--seed` 固定噪声；`--fps` 仅控制输出视频播放速度，不改变预测采样索引，也不意味着均匀的仿真时间间隔。

## 训练入口与恢复

启动脚本默认加载 **baseline**，所以正式 clean 配置必须显式指定。该脚本默认选择 GPU 4–7、4 个进程；运行前按实际空闲资源设置 `CUDA_VISIBLE_DEVICES` 和 `NPROC_PER_NODE`。

```bash
# 只检查正式配置。--allow-batch-change 允许实际 batch 128 与参考值 832 不同。
bash scripts/train_obj_video.sh \
  --config configs/obj_video_train_v1.yaml --allow-batch-change --dry-run

# 新训练使用新的输出目录；此命令会真正启动训练。
bash scripts/train_obj_video.sh \
  --config configs/obj_video_train_v1.yaml --allow-batch-change \
  output_dir=runs/obj_video_train_new
```

**训练检查和新训练还需要额外准备** `weights/checkpoints_video/vpp2-video-stage2-17f.pth`。训练的 `--dry-run` 也会检查该初始化文件是否存在；只下载 clean 10k 权重不足以运行上述训练命令。使用 clean 10k 做推理时则不需要这份 Stage-2 初始化权重。

上面的配置从 Stage-2 初始化。若改为从训练后的权重开始新训练，须显式设置相应初始化参数，并重新决定优化器和调度器。

| 用途 | 配置字段 | 所需材料 |
| --- | --- | --- |
| 从权重开始新训练 | `resume_ckpt=...` | DiT 权重；创建新的优化器 / 调度器 |
| 严格接续原训练 | `resume=...` | 完整 trainer state，含优化器 / 调度器 / RNG 等 |

两种恢复方式互斥。HF 的 `step_010000.pt` 不能单独恢复完整 trainer state。备份里的训练日志证明原训练曾保存完整状态，但不代表该状态已经上传。

## 评测与对照展示

批量评测和步数扫描脚本位于 [scripts/](../scripts/)，例如 [evaluate_obj_video_full.py](../scripts/evaluate_obj_video_full.py)。使用前先检查各脚本的配置、清单和输出路径；历史评测选择与生成视频未随代码仓库完整保存。

[build_obj_video_review.py](../scripts/build_obj_video_review.py) 接受已有批量推理结果：目录须含 `batch_trace.json` 和 `samples/*/{trace.json,future.mp4}`。它不能直接读取上面单窗口 CLI 的输出目录。

```bash
CUDA_VISIBLE_DEVICES='' .venv-train/bin/python scripts/build_obj_video_review.py \
  --run runs/obj_video_step4000_motion_start_v1
```

上述路径是原服务器上的历史批量结果示例，需替换为实际已完成的 run。默认左侧显示按精确未来索引提取的全景 RGB，右侧显示物体预测；每个面板仍为 observer 左 / ego 右。未来 RGB 仅用于推理后展示，数值指标仍与物体 GT 比较。使用 `--left-source object` 可改成物体 GT 对照；用 `--output` 指定新的输出目录，已有目录不会被覆盖。

## 代码检查

恢复依赖后可运行相关测试：

```bash
source env_train.sh
python -m unittest discover -s tests -p 'test_obj_video_*.py' -v
```

这些测试与历史单窗口验证各有覆盖范围。训练完成、可恢复权重和模型泛化效果应分别依据训练日志、备份校验和对应评测来判断。
