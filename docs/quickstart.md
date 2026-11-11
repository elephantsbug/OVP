# OVP Quickstart

[项目首页](../README.md) · [语言指令 Demo](inference_demos.md) · [完整训练与推理说明](robotwin_clean_training.md)

先看效果可直接打开 [RoboTwin random 视频](../assets/demos/robotwin_random_direction_demo.mp4) 或 [CES 视频](../assets/demos/cse_direction_demo.mp4)，无需下载权重。

本页以 **RoboTwin clean 10k** 为最小推理示例：9 帧全景 RGB 条件和缓存的英文指令 embedding，生成 16 帧黑背景物体视频。当前仓库发布的配置支持 observer 左、ego 右的 **448×224** 输入。

| 模型 | 下载与使用范围 |
| --- | --- |
| RoboTwin clean 10k | 按下文准备环境、权重和本地样本后运行 |
| RoboTwin random 10k | [权重与配置](robotwin_random_step10000_checkpoint.md)；原运行时的源码修改未完整发布 |
| CES 5k | [权重与评测](cse_step5000_checkpoint.md)；需要另行准备单 observer、640×480 的兼容代码与配置 |

random / CES 的 Demo 可直接观看；仅替换下文的 checkpoint 路径，不能完整复现它们的展示。下载地址、文件名和服务器历史路径保留原有标识。

## 1. 获取代码与环境

使用 Linux、Python 3.10 和支持 BF16 的 NVIDIA GPU。推理需容纳完整 DiT、冻结组件和激活；本文未给出经过测量的最低显存。单份 DiT 权重为 **32.79 GB**，另需冻结组件和输入数据的磁盘空间。

```bash
git clone https://github.com/elephantsbug/OVP.git
cd OVP

# 恢复已包含本项目修改的上游源码；保留已有源码目录。
if [ ! -e third_party/vpp2 ]; then
  tar -xzf backups/step_010000_20261026/vpp2-source.tar.gz
fi

python3.10 -m venv .venv-train
source .venv-train/bin/activate
python -m pip install --upgrade pip
```

下面沿用备份记录的 CUDA 13.0 / PyTorch 组合，要求驱动支持该构建；其他驱动环境需自行选择匹配的 PyTorch，并同步调整版本约束。这里只安装推理依赖，训练环境另见 [恢复指南](../backups/step_010000_20261026/README.md)。

```bash
python -m pip install torch==2.11.0 torchvision==0.26.0 \
  --index-url https://download.pytorch.org/whl/cu130
python -m pip install -r third_party/vpp2/requirements.txt \
  -c third_party/vpp2/environment-reference.txt
python -m pip install huggingface_hub==0.36.2
python -m pip check

export VPP2_PROJECT_ROOT="$PWD"
export PYTHONPATH="$PWD/src:$PWD/third_party/vpp2/src"
export TOKENIZERS_PARALLELISM=false
python -m vpp2_obj.infer --help
```

后续命令均在仓库根目录、同一个已激活的 shell 中运行。新终端需重新激活环境并设置上述变量。`env_train.sh` 中的 CUDA 路径来自原服务器；新机器使用这里的设置，或先按本机环境修改该脚本。解包后的源码已包含补丁，无需再次应用 `upstream.patch`。

## 2. 下载 clean 权重与冻结组件

```bash
hf download elephantsbug/vpp2-obj-generation-step10000 step_010000.pt \
  --revision fea338d4a21fa7c296d7d3729be39b0720d8ea4d \
  --local-dir weights/robotwin-clean-step10000

echo '95b8cfe616e06e5456cc94f31fa0e0da880a020e1c4a81f86e3d0123af694858  weights/robotwin-clean-step10000/step_010000.pt' \
  | sha256sum -c -

hf download Haodong082399/VPP2 \
  --revision 04824ddb3aae11f13db50984ed7d540c3b77dd9f \
  --local-dir weights \
  --include 'checkpoints/Wan2.1-I2V-14B-480P/config.json' \
            'checkpoints/Wan2.1-I2V-14B-480P/Wan2.1_VAE.pth' \
            'checkpoints/Wan2.1-I2V-14B-480P/models_clip_open-clip-xlm-roberta-large-vit-huge-14.pth'
```

预期 SHA-256 检查输出 `OK`。上述推理使用已缓存的文本 embedding，因此不加载 T5；新指令需要额外下载 T5/tokenizer 并重新编码，见下文。指定已训练 checkpoint 后，无需下载 Stage-2 初始化权重。

## 3. 准备一个 clean 输入窗口

先按 [数据与 prompt 指南](robotwin_clean_data.md) 准备 clean 清单、RGB 视频、去重索引和文本缓存。**GitHub 的清单快照只保存元数据，未附带所引用的视频、NPZ 或 embedding，因此新 clone 不能直接运行样本推理。**

| 必要输入 | 要求 |
| --- | --- |
| JSONL 清单 | 默认 `data/robotwin_observer_ego_obj_splits_v1/test.jsonl`；也可通过 `--manifest` 指定已准备好的清单 |
| RGB 视频 | `condition_video_path` 指向实际视频；每帧高 224、宽 448，observer 左、ego 右 |
| 去重索引 | `dedup_index_path` 指向有效 NPZ，索引、时间戳、帧数和 SHA-256 与清单一致 |
| 文本 embedding | 与该行 `prompt`、哈希和缓存签名匹配；BF16 `context [128,4096]`、bool mask `[128]`、int64 `input_ids [128]`；文件名为 `<prompt SHA-256>.pt` |

清单中的相对媒体路径相对于清单所在目录解析。迁移原数据时，需更新清单内的绝对路径及相关缓存约定；设置项目根目录不会自动改写它们。推理会检查清单中所有行的索引，可以提供只含所选 episode 的有效清单来减少所需索引文件。

下文使用 clean 测试集的 `adjust_bottle / episode_0000016`、锚点 `98`。换样本时同时修改 task、episode 和 anchor；anchor 是**去重后的序号**。历史不足用 episode 首帧补齐。推理读取 RGB 首帧与历史，不读取目标物体视频；清单仍须保留与 RGB 路径不同的 `target_video_path` 字段。

## 4. 检查并生成视频

先做 CPU 检查；输出目录须尚不存在：

```bash
python -m vpp2_obj.infer \
  --config configs/obj_video_train_v1.yaml \
  --checkpoint weights/robotwin-clean-step10000/step_010000.pt \
  --manifest data/robotwin_observer_ego_obj_splits_v1/test.jsonl \
  --task adjust_bottle --episode episode_0000016 --anchor 98 \
  --steps 10 --seed 0 --fps 6 \
  --output runs/quickstart_clean --dry-run
```

预期终端 JSON 中出现 `"status": "dry_run_passed"`。此步骤检查 checkpoint 文件存在、配置、清单和去重索引，不解码 RGB、不加载模型或 embedding，也不写输出，不能代替实际 GPU 推理检查。

选择本机空闲 GPU 后运行（下面以物理 GPU 0 为例）：

```bash
CUDA_VISIBLE_DEVICES=0 python -m vpp2_obj.infer \
  --config configs/obj_video_train_v1.yaml \
  --checkpoint weights/robotwin-clean-step10000/step_010000.pt \
  --manifest data/robotwin_observer_ego_obj_splits_v1/test.jsonl \
  --task adjust_bottle --episode episode_0000016 --anchor 98 \
  --steps 10 --seed 0 --fps 6 --device cuda:0 \
  --output runs/quickstart_clean
```

成功后终端返回 `"status": "complete"`，新目录包含：

- `future.mp4`：16 帧物体预测；`--fps 6` 控制播放速度。
- `trace.json`：实际 prompt、输入索引、checkpoint、seed 和推理参数。

再次运行请使用新的 `--output` 目录。上述 clean 示例使用配置默认的 10 步去噪；首页的 random / CES 展示使用 8 步，其数据与运行环境见 [Demo 说明](inference_demos.md)。

## 更换语言指令

当前 CLI 从清单读取指令与文本缓存，没有 `--prompt` 或 `--video` 参数。更换指令时，需先用 [缓存脚本](../scripts/cache_obj_prompt_embeddings.py) 和冻结 UMT5 生成对应 embedding，再更新清单的 prompt、embedding 路径、哈希与缓存签名；仅修改 prompt 字符串会触发一致性检查。完整流程见 [Prompt 与文本缓存](robotwin_clean_data.md#prompt-与文本缓存)。

做左右指令对照时固定 RGB 历史、anchor、checkpoint、seed 和去噪步数，只修改方向描述。结果用于观察语言条件的影响；定性 Demo 的输入、选择方法与限制见 [语言指令 Demo](inference_demos.md)。

本页验证记录（2026-11-11）：已用发布的 clean 源码和 H200 上现有数据通过上述样本的 CPU dry-run，并核对命令参数与下载路径；本次文档更新未重新安装完整环境或执行 GPU 推理。
