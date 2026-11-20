# KINA — 诺亚方舟知识指数评测

[English](README.md)

![KINA](images/kina_banner_white.svg)

KINA 包含 899 道单选题，覆盖 261 个学科。每题只有一个正确选项。本仓库通过 OpenAI 兼容的 Chat Completions 接口运行公开数据集，并在本地按选项字母计分。[官方 KINA 仓库](https://github.com/2077AI/KINA) · [官方数据集](https://huggingface.co/datasets/2077AIDataFoundation/KINA)

## 安装和下载数据

在 Bash 或 Zsh 中执行以下命令。需要 Git、curl 和 Python 3.11。

```bash
git clone https://github.com/jackhe183/KINA.git
cd KINA
python3.11 -m venv .venv
.venv/bin/python -m pip install -e .
mkdir -p data
curl -fL --retry 3 \
  https://huggingface.co/datasets/2077AIDataFoundation/KINA/resolve/860c64e856375ed42929737fb8b8f37272c97065/KINA.json \
  -o data/KINA.json
```

CLI 读取本地 JSON，并使用每条记录的 `index` 作为题目 ID。公开数据集的 ID 范围为 0–898。

## 流式试跑 3 题

将 `OPENAI_BASE` 改为你的 API 基础地址，例如以 `/v1` 结尾的地址。将 `KINA_MODEL` 改为该接口接受的模型 ID。按提示输入 API key。

```bash
export OPENAI_BASE='https://your-gateway.example/v1'
KINA_MODEL='gpt-6.1-sol'
printf '请输入 API key（输入时不显示）：'
read -rs OPENAI_KEY
printf '\n'
export OPENAI_KEY

.venv/bin/python src/kina_bench/run_openai_chat.py \
  --model_id "$KINA_MODEL" \
  --data_name KINA \
  --limit 3 \
  --n_sampling 1 \
  --n_thread 20 \
  --requests_per_second 1 \
  --max_tokens 16384 \
  --timeout 300 \
  --stream
```

`--limit 3` 取数据集的前 3 条记录，并将结果写入单独的目录。移除 `--stream` 可使用普通 JSON 响应。如果接口要求 `max_completion_tokens`，把 `--max_tokens 16384` 换成 `--max_completion_tokens 16384`。

所有请求都使用 Chat Completions。请求省略 `temperature`、`top_p` 和推理强度相关参数，由接口应用默认设置。流式请求会要求接口返回 token 用量。

## 全量运行 899 题

在同一个终端中沿用前面的接口和模型配置。执行以下命令，无需指定 `--limit`。

```bash
.venv/bin/python src/kina_bench/run_openai_chat.py \
  --model_id "$KINA_MODEL" \
  --data_name KINA \
  --n_sampling 1 \
  --n_thread 20 \
  --requests_per_second 1 \
  --max_tokens 16384 \
  --timeout 300 \
  --stream
```

每题需要采样 4 次时，如果接口支持单个请求返回 4 个答案，可使用 `--n_sampling 4`。如果接口每次只返回 1 个答案，则独立跑 4 轮全量评测。用相同参数开始下一轮之前，先将上一轮的结果目录移出 `results/` 并归档。计算 AVG@4 时，每题每轮计一次，将四轮答对总数除以 3,596。

## CLI 参数

| 参数 | 默认值 | 用途 |
| --- | --- | --- |
| `--model_id` | 必填 | 发送给接口的模型 ID。 |
| `--data_name` | `KINA-899` | `data/` 下的文件名，不带扩展名。按上文下载的数据用 `KINA`。 |
| `--limit` | 全部记录 | 只读取前 N 条记录，结果写入单独的目录。 |
| `--n_sampling` | `1` | 每题发一个 Chat 请求，要求返回 1、4 或 8 个答案。 |
| `--n_thread` | `1` | 同时处理的题目数上限。 |
| `--requests_per_second` | `1` | 限制当前进程的请求启动速率，重试也计入。支持正小数。 |
| `--max_tokens` | `16384` | 用 `max_tokens` 字段发送 token 上限。 |
| `--max_completion_tokens` | 未设置 | 改用 `max_completion_tokens` 字段发送 token 上限。两个 token 参数只能选一个。 |
| `--timeout` | `300` | 每次请求尝试的超时秒数，包含读取流式响应。 |
| `--stream` | 关闭 | 收齐 Chat Completions 流式响应后计分。 |
| `--skip_inference` | 关闭 | 对已保存的响应进行离线计分。 |
| `--overwrite` | 关闭 | 对当前输出路径从头推理。 |
| `-h`、`--help` | | 查看 CLI 帮助。 |

使用 `--n_thread 20 --requests_per_second 1` 时，首个请求立即启动，之后每次启动至少间隔 1 秒，同时最多处理 20 道题。可重试的错误最多重试 10 次，重试遵循同一速率限制。

## 续跑、计分和查看结果

- **续跑。** 重新执行原命令，复用已保存的完整响应，并重跑未完成的题目。普通运行收齐所选题目后写出最终计分文件。
- **离线计分。** 在原命令末尾加 `--skip_inference`，无需发起 API 请求。沿用原有的模型、数据集、采样数、token 字段和值、流式选项及题目数量限制。未跑完时得到的是部分结果，汇报全量得分前先核对 `total_num`。
- **查看文件。** 结果在 `results/<model_id>/<generation>/`。例如上面的流式试跑写入 `results/gpt-6.1-sol/n1_tokens16384_max_tokens_stream_limit3/`。`KINA.jsonl` 保存原始响应和用量，`KINA.json` 保存逐题得分、提取的选项和标准答案。

评分读取每条可见回答的最后一个非空行，识别 `Answer: X` 中的一个有效选项。选项正确得 1 分，缺失、含糊或错误得 0 分。一题有多个回答时，题目得分为答对的比例。汇总得分是各题得分的平均值。

```bash
.venv/bin/python src/kina_bench/pretty_print.py --data_name KINA
```

## 运行离线测试

测试使用本地模拟的 Chat 接口和小型数据样例，覆盖流式响应、请求参数、限速、超时处理和答案提取。无需 API key 或官方数据集。

```bash
.venv/bin/python -m unittest discover -s tests -v
```
