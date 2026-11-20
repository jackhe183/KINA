# KINA — Knowledge Index of Noah's Ark Benchmark

[中文文档](README.zh-CN.md)

![KINA](images/kina_banner_white.svg)

KINA evaluates model knowledge with 899 multiple-choice questions across 261 disciplines. Each question has one correct option. This fork runs the public dataset through an OpenAI-compatible Chat Completions endpoint and scores the submitted option letter locally. [Original KINA repository](https://github.com/2077AI/KINA) · [Official dataset](https://huggingface.co/datasets/2077AIDataFoundation/KINA)

## Install and download the dataset

Run these commands in Bash or Zsh with Git, curl, and Python 3.11.

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

The CLI reads the local JSON. It uses each row's `index` as the question ID. The public dataset contains IDs 0–898.

## Run a three-question streaming test

Set `OPENAI_BASE` to your endpoint's API base URL, for example one ending in `/v1`. Set `KINA_MODEL` to the model ID accepted by that endpoint. Enter the API key at the prompt.

```bash
export OPENAI_BASE='https://your-gateway.example/v1'
KINA_MODEL='gpt-6.1-sol'
printf 'Enter API key (hidden): '
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

`--limit 3` selects the first three rows and stores the results in a separate directory. Remove `--stream` to use a regular JSON response. If your endpoint requires `max_completion_tokens`, replace `--max_tokens 16384` with `--max_completion_tokens 16384`.

The runner always uses Chat Completions. It omits `temperature`, `top_p`, and reasoning controls so the endpoint can apply its defaults. Streaming requests ask the endpoint to include token usage.

## Run all 899 questions

Use the same endpoint and model configuration in the same shell. Run this command without `--limit`.

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

For four samples per question, use `--n_sampling 4` on an endpoint that supports four choices in one response. If your endpoint accepts only one choice, use four independent full runs with separate result directories. Move each run's result directory out of `results/` before starting the next run with the same options. For AVG@4, add the correct answers across the four runs and divide by 3,596, counting each question once per run.

## CLI options

| Option | Default | Behavior |
| --- | --- | --- |
| `--model_id` | Required | Model ID sent to the endpoint. |
| `--data_name` | `KINA-899` | Dataset basename under `data/`. Use `KINA` for the downloaded file above. |
| `--limit` | All rows | Read the first N rows and use a separate output directory. |
| `--n_sampling` | `1` | Request 1, 4, or 8 choices per question in one Chat request. |
| `--n_thread` | `1` | Set the maximum number of concurrent question workers. |
| `--requests_per_second` | `1` | Limit request starts per process, including retries. Accepts positive decimals. |
| `--max_tokens` | `16384` | Send the token budget as `max_tokens`. |
| `--max_completion_tokens` | Unset | Send the token budget as `max_completion_tokens` instead. Choose only one token option. |
| `--timeout` | `300` | Set the deadline in seconds for each attempt, including stream consumption. |
| `--stream` | Off | Collect streamed Chat Completions chunks before scoring. |
| `--skip_inference` | Off | Score saved responses locally. |
| `--overwrite` | Off | Start inference from scratch for the selected output path. |
| `-h`, `--help` | | Show CLI help. |

With `--n_thread 20 --requests_per_second 1`, the first request starts immediately. The runner waits at least one second between request starts and allows up to 20 questions to be in flight. Retryable failures can receive up to ten retries, all subject to the same start-rate limit.

## Resume, score, and inspect results

- **Resume.** Rerun the same command to reuse saved completions and retry unfinished questions. A normal run writes the final score file after results cover the selected questions.
- **Score locally.** Append `--skip_inference` to the same command to score saved responses without an API request. Keep the model, dataset, sample count, token field and value, stream setting, and question limit. An incomplete run gives a partial score. Check `total_num` before reporting a full benchmark score.
- **Read the files.** Find results under `results/<model_id>/<generation>/`. For example, the streaming smoke test writes to `results/gpt-6.1-sol/n1_tokens16384_max_tokens_stream_limit3/`. `KINA.jsonl` contains raw responses and usage. `KINA.json` contains per-question scores, extracted choices, and reference answers.

The last nonempty line of each visible response must contain one listed option in the form `Answer: X`. A matching letter earns 1. Missing, ambiguous, or incorrect answers earn 0. With multiple completions, the question score is the fraction that are correct. The summary averages the question scores.

```bash
.venv/bin/python src/kina_bench/pretty_print.py --data_name KINA
```

## Run the offline tests

The tests use a local mock Chat endpoint and a small fixture dataset. They cover streaming, request options and pacing, timeout handling, and answer extraction. The test command requires neither an API key nor the official dataset.

```bash
.venv/bin/python -m unittest discover -s tests -v
```
