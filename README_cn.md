# longds-bench

[English](README.md) | [简体中文](README_cn.md)

一个可移植、**与智能体无关（agent-agnostic）** 的技能（skill），用于运行 **LongDS-Bench**（[zjunlp/DataMind](https://github.com/zjunlp/DataMind/tree/main/longds)）——长程、多轮的 agentic 数据分析基准——**并且让智能体自身充当被测的 runtime**。

与官方 harness 不同，本 skill **不**使用 DSGym 那约 12 GB 的 Docker 执行器，也**不**通过 LiteLLM 去驱动某个模型。相反，加载本 skill 的智能体*本身*就是被测对象：它读取数据集，用自己的工具和一个持久 Python 会话完成多轮分析，逐轮给出答案，然后由官方的 LLM-judge 规则打分。任何带有 shell / 代码执行工具的 harness（Anda Bot 或其他）都能使用它。

> **注意：** LongDS 数据集约 **19.5 GB**。由你作为操作者**一次性**下载并预处理；智能体本身从不下载任何东西。本方案得到的分数仅供**参考**，**不**等同于官方排行榜可比分数（执行环境与 agent 脚手架都与论文不同）。

---

## 测什么

68 个任务 / 2,225 轮，覆盖六个领域（Business、Community、Education、Geoscience、Social Good、Sports）。每个任务是一段连续的多轮对话，其中分析状态会不断演化（状态继承、更新、反事实扰动、回滚、多状态组合）。每一轮按 **0/1** 判分；所有轮的均值即为准确率。

论文参考值（官方 runtime）：最佳 ≈ **48.45**（Gemini-3.1-Pro），GPT-5.4 **43.50**，Claude-4.6-Sonnet **41.56**。

## 本 skill 包含什么

| 文件                         | 由谁运行 | 用途                                                                                      |
| ---------------------------- | -------- | ----------------------------------------------------------------------------------------- |
| `SKILL.md`                   | 智能体   | 规则 + 智能体对自身执行的逐轮循环。                                                       |
| `scripts/prepare_dataset.py` | 操作者   | 把已下载的数据集拆分为「剥离答案的智能体 **manifest**」+「留存的 **gold**」。**不下载**。 |
| `scripts/pysession.py`       | 智能体   | 每个任务一个持久 IPython 会话（跨步、跨轮保持状态）。                                     |
| `scripts/judge.py`           | 操作者   | 官方二元 LLM-judge（逐字复用 `JUDGE_PROMPT`）；报告总体与各领域准确率。                   |

---

## 准备（操作者，一次性）

克隆本仓库并选择一个工作目录。本仓库目录本身就是 `SKILL_DIR`；不需要把它安装或注册成某个具名 skill。

```bash
mkdir -p "$HOME/github"
git clone https://github.com/ldclabs/longds-bench.git "$HOME/github/longds-bench"

export SKILL_DIR="$HOME/github/longds-bench"          # 包含 SKILL.md + scripts/ 的目录
export WORK="$HOME/longds-bench-work"
export STAGING="$WORK/dataset"                        # 原始下载（约 19.5 GB）
export LONGDS_RUN="$WORK/run"                          # 智能体使用的预处理工作区
export LONGDS_VENV="$WORK/venv"
mkdir -p "$WORK"
```

### 1. Python 环境（替代 DSGym 的 Docker 镜像）

```bash
uv venv "$LONGDS_VENV" --python 3.12
. "$LONGDS_VENV/bin/activate"
uv pip install ipykernel jupyter_client openai huggingface_hub \
  pandas numpy scipy scikit-learn statsmodels python-dateutil pyarrow openpyxl
# 部分任务会用到更多库；当某一步报 ModuleNotFoundError 时按需安装即可。
```

### 2. 下载数据集（这一步由你做——智能体永远不做）

**完整基准（约 19.5 GB）：**

```bash
hf download zjunlp/LongDS --repo-type dataset --local-dir "$STAGING"
```

**或先下子集快速试跑**（单个领域——`*` 可跨 `/` 匹配）：

```bash
hf download zjunlp/LongDS --repo-type dataset --local-dir "$STAGING" \
  --include "task/longds/task_list.json" "task/longds/business/*" "data/longds/business/*"
```

### 3. 预处理 = 剥离答案 + 构建智能体工作区

`task.json` 内联了参考 `answer` 和参考解题 `code`。这一步把它们抽取到**留存的 gold** 文件，并给智能体一份**无答案的 manifest**。

```bash
# 先跑一个冒烟切片：
python "$SKILL_DIR/scripts/prepare_dataset.py" \
  --dataset-root "$STAGING" --out-dir "$LONGDS_RUN" \
  --task-limit 1 --turn-limit 3

# 全量（可选按领域）。--strip-source 还会从下载目录里删除 task.json/task.py/task.ipynb，
# 使答案的唯一副本只存在于 $LONGDS_RUN/gold：
python "$SKILL_DIR/scripts/prepare_dataset.py" \
  --dataset-root "$STAGING" --out-dir "$LONGDS_RUN" --strip-source

cat "$LONGDS_RUN/index.json"   # 检查：任务数、各任务轮数，以及 data_dir_exists == true
```

产物结构：

```
$LONGDS_RUN/
├── index.json          # 任务清单
├── manifest/<key>.json  # 智能体读取：turn_id / context / question / data_dir（无答案）
├── gold/<key>.json      # 仅 judge：question + ground_truth（留存）
└── answers/             # 智能体把答案写到这里，每个任务一个 <key>.json
```

---

## 让智能体来跑

让你的智能体读取并遵循本仓库里的 `SKILL.md`，并把预处理工作区指给它。不需要安装 `longds-bench` skill。例如：

> `SKILL_DIR=/path/to/longds-bench`，`LONGDS_RUN=/path/to/run`，`LONGDS_VENV=/path/to/venv`。请按照 `/path/to/longds-bench/SKILL.md` 的指示运行 LongDS-Bench 测试任务。先跑单任务试点，把分数给我看，然后在全量运行前先问我。

如果只需要让智能体答题、不需要当场评估分数，也可以明确写成：

> `SKILL_DIR=/path/to/longds-bench`，`LONGDS_RUN=/path/to/run`，`LONGDS_VENV=/path/to/venv`。请按照 `/path/to/longds-bench/SKILL.md` 的指示运行测试，使用最高推理能力获得最好结果。你只需要答题，不需要评估分数。为了节省时间，可并发运行 3～5 个任务。

随后智能体按 `SKILL.md` 执行：对每个任务，在该任务的 `data_dir` 处打开一个持久 Python 会话，按 `turn_id` 顺序逐轮以连续代码执行求解（不画图；数值答案要精确），并把每一轮的最终答案追加写入 `answers/<key>.json`。**诚信约束：** 智能体只能读 `manifest/`，绝不能读 `gold/` 或原始 `task.json`。

全量运行可能涉及数千个推理步、耗时数小时——最好用支持后台执行、且能「每个任务一个 worker/子智能体」的 harness。

## 打分

使用一个 judge 端点（OpenAI 兼容）。**judge 模型应与被测智能体的模型不同**，以避免自评偏差。

```bash
. "$LONGDS_VENV/bin/activate"
export JUDGE_API_KEY="<key>"; export JUDGE_BASE_URL="https://api.deepseek.com"
python "$SKILL_DIR/scripts/judge.py" \
  --answers "$LONGDS_RUN/answers" --gold "$LONGDS_RUN/gold" \
  --out "$LONGDS_RUN/results_eval.json" --judge-model "deepseek-v4-pro" --max-workers 8
```

输出（`results_eval.json` + 控制台）：总体准确率、各领域准确率、各任务准确率，以及逐轮的分数/理由。空答案的轮计 0；judge 重试 3 次仍无法解析的轮会被排除并报告为「未判分」。

---

## 注意事项

- **仅供参考，非官方。** 本地 venv ≠ DSGym 固定的 Docker 镜像；智能体自己的循环 ≠ 基准固定的 ReAct 脚手架。把分数当作该智能体能力的指示值，而非排行榜分数。
- **Judge 偏差。** 自评会抬高分数；优先用独立的 judge 模型，并注明用的是哪个。
- **成本/耗时。** 全量 = 数千个付费步骤 + 2,225 次 judge 调用。务必先试点。

## 鸣谢与许可

- **本仓库自有代码**（此处编写的脚本与文档）—— MIT，© 2026 LDC Labs（见 [LICENSE](LICENSE)）。
- **LongDS-Bench**（基准、多轮任务协议、`JUDGE_PROMPT`）—— 由 Xu 等人（2026）创作，[zjunlp/DataMind](https://github.com/zjunlp/DataMind)，**Apache-2.0**。`scripts/judge.py` **逐字**复用其 `JUDGE_PROMPT`，`SKILL.md` 遵循其任务协议，以使分数与官方规则一致。该部分内容版权归原作者所有、依 Apache-2.0 授权——见 [NOTICE](NOTICE)。
- **DSGym** —— [fannie1208/DSGym](https://github.com/fannie1208/DSGym)（[论文](https://arxiv.org/abs/2601.16344)）。仅在设计上参考，未内置。本适配器有意**不**使用 DSGym 的 Docker runtime。
- **LongDS 数据集** —— [zjunlp/LongDS](https://huggingface.co/datasets/zjunlp/LongDS)，许可为 **`other`**。此处不再分发；你需自行从 Hugging Face 下载，并遵守其条款。

如果你使用 LongDS-Bench，请引用原作者论文（BibTeX 见 [NOTICE](NOTICE)）。
