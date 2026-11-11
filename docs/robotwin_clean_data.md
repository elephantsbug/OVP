# RoboTwin clean：数据与 prompt

[返回项目首页](../README.md) · [训练与推理](robotwin_clean_training.md) · [恢复指南](../backups/step_010000_20261026/README.md)

本文说明 clean 10k 模型使用的 **448×224 observer / ego 双视角** 数据。CES 单视角的数据、划分和时间间隔见 [CES checkpoint 文档](cse_step5000_checkpoint.md#输入输出与数据)。

## 数据约定

输入为全景 RGB episode 首帧、历史窗口和英文动作计划；输出与监督为 `video_object_full`，保留目标物体 RGB 和黑背景。RGB / obj 两路使用相同的视角布局与帧索引。

| 项目 | 约定 |
| --- | --- |
| 单帧布局（宽 × 高） | observer 左 224×224，ego 右 224×224，共 448×224 |
| 空间处理 | 不翻转、不裁剪，复用 VPP2 `training_exact` tensor bilinear resize，`antialias=True` |
| 编码 | MP4 为 `libx264` CRF 20 有损编码；首帧 PNG 保存编码前像素 |
| 时间处理 | 保留源 MP4 帧序、编码帧率和仿真时间戳；转换阶段不做训练抽帧或 FPS 重采样 |
| 转换记录 | 2500 episodes，1,046,361 帧，历史记录为零失败 |
| 配对字段 | 见下表；不能仅把全景 `video_path` 改为物体视频路径 |

机器可读约定见 [obj_video_data_contract.json](../configs/obj_video_data_contract.json)。转换完成记录来自 [原 README 记录](history/readme_20261110.md#paired-object-video)；原始完整转换报告位于服务器本地 `data/robotwin_observer_ego_obj_full_vpp2/validation.json`。

| 配对字段 | 含义 |
| --- | --- |
| `condition_video_path` | 全景 RGB 视频，历史从此读取 |
| `condition_episode_first_frame_path` | 全景 RGB episode 首帧 |
| `target_video_path` | obj 预测监督视频 |
| `target_first_frame_path` | obj 首帧，仅供目标检查，不能作为 RGB 条件图 |
| `prompt` | 根据视频审阅后整理的英文指令 |
| `prompt_review_flags` | 保留待复核问题 |
| `dedup_index_path` / `dedup_index_sha256` | 共享的去重索引及哈希 |
| `text_embedding_path` | 与该行 prompt 版本匹配的文本缓存 |

## 准备 RGB / obj 视频

已有原始 RoboTwin clean 数据、上游源码和转换依赖后，在项目根目录执行：

```bash
# observer / ego 全景 RGB
bash scripts/convert_observer_ego.sh --dry-run
bash scripts/convert_observer_ego.sh --workers 16

# 同布局的物体视频与配对清单
bash scripts/convert_observer_ego_obj_full.sh --dry-run
bash scripts/convert_observer_ego_obj_full.sh --workers 16
```

RGB 输出位于服务器本地 `data/robotwin_observer_ego_vpp2/`，obj 输出位于 `data/robotwin_observer_ego_obj_full_vpp2/`。更早的 `scripts/convert.sh` 生成 **416×240 单 observer 带黑色侧栏** 数据，不是上述 clean 双视角训练输入；其步骤保留在 [历史转换流程](history/readme_20261110.md#legacy-observer-conversion)。

物体输出目录的结构如下：

```text
videos/<task>/<episode>.mp4       # obj 监督视频
images/<task>/<episode>.png       # obj 首帧，非 RGB 条件
timestamps/<task>/<episode>.csv   # 仿真时间戳
records/<task>/<episode>.json     # 来源、RGB 引用、哈希与对齐检查
manifest.jsonl                   # 转换成功记录
paired_all.jsonl                 # 全量 RGB / obj / prompt 配对
paired_no_flags.jsonl            # 排除待复核 prompt 的历史配对
paired_needs_review.jsonl        # 待复核配对
summary.json                     # 数量、帧数、失败与配置
```

转换会逐帧解码并核对两路 obj 的相对 PTS、仿真时间戳、RGB / obj 帧数与 FPS、原相机尺寸，并检查 RGB 引用文件及源文件哈希。源文件在转换期间发生变化会被拒绝；原始媒体不修改。

每个输出目录有独立标记与锁。续跑先校验已有输出再跳过；按 `--task` 或 `--limit` 重跑会把汇总索引改成所选范围，预览应使用独立 `--output-root`。完整目录的汇总应使用全量命令生成。

## Train / test 划分

| 集合 | Episodes | 用途 |
| --- | ---: | --- |
| train | 2155 | 优化与任何拟合的数据统计 |
| test | 240 | 同任务下的新 episode 评估，不参与训练 |
| 排除 | 105 | prompt 待复核，原始媒体与文本仍保留 |

2395 条可用 episode 按任务分层、固定 seed 42，以完整 episode 为单位作 90% / 10% 划分。两个集合都覆盖全部 50 个任务；同一 episode 的视角、帧与窗口保持同一归属。这不是未见任务泛化评测。

实际运行使用本地 `data/robotwin_observer_ego_obj_splits_v1/train.jsonl` 和 `test.jsonl`。**不要用未划分的 `paired_all.jsonl`、`paired_no_flags.jsonl` 或 prompt 总目录启动训练。**

GitHub 可查看冻结快照：[train](../backups/step_010000_20261026/metadata/data/robotwin_observer_ego_obj_splits_v1/train.jsonl)、[test](../backups/step_010000_20261026/metadata/data/robotwin_observer_ego_obj_splits_v1/test.jsonl)、[排除清单](../backups/step_010000_20261026/metadata/data/robotwin_observer_ego_obj_splits_v1/excluded_prompt_review.jsonl)、[划分来源与数量](../backups/step_010000_20261026/metadata/data/robotwin_observer_ego_obj_splits_v1/summary.json)、[验证记录](../backups/step_010000_20261026/metadata/data/robotwin_observer_ego_obj_splits_v1/validation.json)。这些是元数据，不包含引用的视频、NPZ 和 embedding。

任务间测试配额按最大余数分配，总计 240；任务内按 seed 42 的稳定 SHA-256 排序选择。行中保留媒体字段，并记录源帧数、去重帧数和 `sampling_index_space`。

```bash
python3 scripts/split_obj_video_dataset.py --test-fraction 0.10 --seed 42 --dry-run
```

这个命令复现最初划分及其历史长 prompt，不直接重建后来的短 prompt 缓存。实际输出默认拒绝覆盖已有目录；新版本应使用新的 `--output-root`，再明确切换训练配置。

## Prompt 与文本缓存

clean 训练使用 `prompt_obj_chronological_128_v1`：2395 条按视频动作时序整理的英文 prompt，包含固定的 RGB 条件 / 黑背景物体预测前缀，完整文本连同特殊 token 为 57–120 个 UMT5 token（中位数 90），在 128 长度内无截断。它沿用已完成的视频审阅，不代表又进行了一轮视觉审阅。

本地 prompt 总目录为 `runs/prompt_obj_chronological_128_v1/exports/paired_training.jsonl`，tokenizer 快照位于同一 run 的 `tokenizer/`。`text_prompt_enhanced.txt` 使用 `prompt@@images/<task>/<episode>.png` 格式，`by_task/` 中的图片路径也相对于 exports 或 RGB 数据根目录。

完整 prompt 只编码一次：不附加旧视角后缀、重复前缀或 `@@images/...`。视频中的 observer image-left 对应实际右臂，整理文本不改变物理臂身份。转换时的旧长 prompt 和原始配对目录仍作为历史记录保存。

| 缓存项目 | 内容 |
| --- | --- |
| 本地目录 | `data/text_embeddings/prompt_obj_chronological_128_v1_20261019/` |
| 唯一文本 | 1382；相同文本共享冻结 UMT5 结果 |
| 文件命名 | `embeddings/<prompt UTF-8 SHA256>.pt` |
| 每个 PT | BF16 `context [128,4096]`、bool padding mask `[128]`、`input_ids` 和来源哈希 |
| 配套元数据 | `index.jsonl`、`cache_contract.json`、`summary.json`、`status.json` |
| 清单引用 | 文本路径、prompt / 文件哈希、缓存版本和有效 token 数 |

[缓存生成脚本](../scripts/cache_obj_prompt_embeddings.py) 使用隔离训练环境与空闲 GPU；历史命令使用 `CUDA_VISIBLE_DEVICES=4`，运行前需按实际资源选择。`--resume` 只适用于相同约定的已有缓存。`status=complete` 要求逐项通过 token、mask、shape、dtype、有限值和哈希检查，不能复用旧长 prompt 的 embedding。见 [保存的缓存摘要](../backups/step_010000_20261026/metadata/data/text_embeddings/prompt_obj_chronological_128_v1_20261019/summary.json)。冻结文本特征的共享不拟合测试集。

[refresh_obj_split_prompts.py](../scripts/refresh_obj_split_prompts.py) 默认只检查，`--commit` 要求完整新缓存。刷新保留 episode 归属、行顺序、媒体路径和去重字段；旧清单及约定备份在本地 `data/robotwin_observer_ego_obj_splits_v1/revisions/before_chronological_128_20261019/`。版本变更记录见 [prompt_refresh_20261019.json](../backups/step_010000_20261026/metadata/data/robotwin_observer_ego_obj_splits_v1/prompt_refresh_20261019.json)。

## 采样索引与时间

令 `s` 为去重后的当前帧序号：

```text
RGB 条件：I0 + [I(s-210), I(s-180), …, I(s-30), Is]，共 9 帧
obj 未来：[O(s+5), O(s+10), …, O(s+80)]，共 16 帧
训练视频：[9 帧 RGB | 16 帧 obj]，共 25 帧
```

历史窗口是 7 帧过去画面加当前帧；加 episode 首帧后共 9 帧。上游 `condition_history_frames` 包含当前帧，应为 8，并启用 episode 首帧，不能直接填 7。历史负索引用第 0 帧补齐并标记 padding，训练只采未来 16 帧完整存在的窗口。

去重只移除相同仿真时间戳的后续帧，保留每个时间戳第一帧和其他正的短间隔，不构造均匀 25 / 30 Hz 网格。先在去重序列采样，再通过同一 NPZ 的 `source_frame_indices` 映射到 RGB / obj 原始 MP4 帧序号。因此 **stride 30 不保证恰好 1 秒，210 个保留帧间隔不保证恰好 7 秒**；实际跨度应查仿真时间戳，不能仅使用 MP4 的标称 FPS。

训练保留 RGB 前缀 loss 权重 1.0 和原总 loss 归一化，单独记录 RGB 前缀与 obj 未来段的 loss。25 帧对应 7 个 latent 时间位置，9 帧条件对应 3 个；推理联合去噪后只导出 `[9:25]`。实现与历史验证细节见 [基线设计记录](obj_video_baseline_v1.md)，实际训练设置见 [训练与推理](robotwin_clean_training.md)。
