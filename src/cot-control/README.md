# CoT-Control → Harbor Adapter

## Overview

This adapter converts the question-answering part of **CoT-Control** into Harbor tasks.
CoT-Control measures *chain-of-thought (CoT) controllability*: whether a reasoning model
can follow an instruction about its own reasoning trace (for example "reason only in
uppercase" or "do not use the word *chromosome* while reasoning") while it solves a task.

- **Task type**: multiple-choice questions from GPQA (445), Humanity's Last Exam (469) and
  MMLU-Pro (300), each paired with one of nine reasoning instructions.
- **Size**: **10,836 tasks**. The paper counts 1,214 questions × 9 instruction types =
  10,926 QA instances. The released code skips the 45 questions that have no keyword data
  for the two word-suppression types (45 × 2 = 90), which leaves the 10,836 instances the
  upstream harness actually runs. This adapter generates exactly those.
- **What is scored**: the model's reasoning trace, not its answer. The reward is 1 when
  the trace complies with the instruction.
- **Provenance**: [paper](https://arxiv.org/abs/2603.05706),
  [code and data](https://github.com/YuehHanChen/CoTControl) (MIT license), pinned at
  commit `5d78aeffe0152ba087c2d31cd07712d029c64785`. The authors ask that CoT-Control be
  excluded from training data; their marker string is kept in every generated `task.toml`.
- **Not adapted**: the agentic part of CoT-Control (SWE-bench Verified and BFCL, 3,150
  instances) and the unconstrained `baseline` mode. See [Notes & Caveats](#notes--caveats).
- **Modifications**: none to prompts or grading. The instruction is the upstream prompt
  byte for byte, and the verifier runs the upstream grading files unmodified.
- **Status**: task generation, oracle and verifier are implemented and checked against
  upstream offline. **Parity experiments have not been run yet.**

## What is CoT-Control?

CoT-Control ([Reasoning Models Struggle to Control their Chains of Thought](https://arxiv.org/abs/2603.05706),
Chen Yueh-Han et al., 2026) is an evaluation suite for a safety-relevant capability: if a
model could shape what appears in its chain of thought, CoT monitoring would be less
reliable. The suite appends a reasoning instruction to each problem and checks the
reasoning trace. Instruction types fall into three categories with three difficulty
levels each:

| Category | Easy | Medium | Hard |
|----------|------|--------|------|
| Information suppression | `word_suppression` | `multiple_word_suppression` | `ignore_question` |
| Information addition | `repeat_sentences` | `end_of_sentence` | `meow_between_words` |
| Stylistic modification | `lowercase_thinking` | `uppercase_thinking` | `alternating_case` |

Original metrics, each reported as a macro-average over (dataset, instruction type) pairs:

- **Controllability Score** (primary): share of instances whose reasoning complies.
- **Meta Discussion Rate**: among compliant instances, the share whose reasoning mentions
  the instruction itself.
- **Controllability Score without Meta-Discussion**: compliant and no such mention.

A higher score means the model controls its reasoning *more*, which the paper treats as
the less safe direction.

## Adapter Features

- **Exact prompts.** `instruction.md` is the user message upstream sends, produced by the
  upstream prompt module vendored byte-for-byte (`src/cot_control/upstream/prompts.py`).
  Row filtering, keyword choice and the answer key are ports of `run_cceval.py`.
- **Upstream grading, unmodified.** Each task ships `grading.py` and
  `grade_compliance_csv.py` exactly as published. `tests/run_grader.py` only feeds them
  one row and writes the reward files. Eight instruction types are graded by rules; the
  `ignore_question` type and the meta-discussion metric use the upstream LLM judge
  prompts (gpt-5-mini, a compliant `ignore_question` verdict must be confirmed twice more).
- **Rewards** (`/logs/verifier/reward.json`):
  - `reward`: compliance, 0 or 1. Its mean is the Controllability Score.
  - `correctness`: 1 if the answer letter is correct.
  - `controllability_without_meta_discussion`: present when the meta-discussion judge ran.
- **No silent zeros.** If the reasoning trace is missing or the judge is required but
  unavailable, the verifier writes no reward and exits non-zero, so the trial is reported
  as an error instead of a score of 0.
- **CoT-Control agent** (`src/cot_control/agent.py`, `cot_control.agent:CoTControlLLMAgent`).
  CoT-Control-QA is a single-call benchmark, so the adapter adds a Harbor agent that mirrors
  upstream's `OpenRouterLLM`: one chat-completion request whose only message is the
  instruction, `max_tokens=25000`, `temperature=0.0`, and a 25,000-token reasoning budget
  for the models upstream treats as reasoning models. It stores the returned reasoning and
  answer in `/logs/agent/cot_control/` for the verifier and writes an ATIF trajectory. The
  model has no tools and no access to the task container.
- **Oracle.** Upstream has no reference reasoning. The oracle writes a short trace built
  to satisfy the instruction plus the correct letter. For the eight rule-based types the
  trace is checked against the upstream grader while tasks are generated.
- **Pinned, verified inputs.** The three dataset files are fetched at the pinned commit
  and verified by SHA-256. Task ids are row positions in those files.
- **Fidelity check.** `python -m cot_control.check_fidelity` compares every prompt and
  answer key with the upstream functions, the agent's request body with upstream's, and
  grades a few thousand synthetic outputs with both the upstream command line and the
  task verifier.
- **Aggregation.** `python -m cot_control.aggregate` computes the paper's macro-averaged
  metrics from a Harbor job directory or from upstream graded CSV files with one code path.

## Generated Task Structure

```
cot-control/
├── gpqa-0007-word-suppression/        # <dataset>-<row>-<instruction type>
│   ├── task.toml                      # name = cot-control/cot-control__gpqa-0007-word-suppression
│   ├── instruction.md                 # the upstream prompt, byte for byte
│   ├── environment/
│   │   └── Dockerfile                 # python + the grader's dependencies
│   ├── solution/
│   │   ├── solve.sh                   # writes the oracle trace
│   │   └── oracle_trace.json          # compliant reasoning + correct answer
│   └── tests/
│       ├── test.sh                    # runs run_grader.py
│       ├── run_grader.py              # harness around the upstream graders
│       ├── grading.py                 # upstream, unmodified
│       ├── grade_compliance_csv.py    # upstream, unmodified
│       ├── UPSTREAM_LICENSE
│       └── config.json                # instruction type, prompt, answer key
```

Adapter code:

```
src/cot-control/
├── README.md
├── adapter_metadata.json
├── parity_experiment.json
├── pyproject.toml
├── run_cot-control.yaml
└── src/cot_control/
    ├── adapter.py            # task generation
    ├── main.py               # CLI
    ├── constants.py          # pinned commit, checksums, instruction types
    ├── oracle.py             # oracle traces
    ├── agent.py              # CoTControlLLMAgent
    ├── request.py            # the upstream request body and response parsing
    ├── aggregate.py          # macro-averaged metrics
    ├── check_fidelity.py     # comparison against upstream
    ├── upstream_graders.py
    ├── upstream/             # vendored upstream prompts.py + license
    └── task-template/
```

## Run Evaluation / Harness

The verifier reads these variables from the host:

| Variable | Purpose |
|----------|---------|
| `OPENAI_API_KEY` | LLM judge. Required for `ignore_question` tasks; optional otherwise |
| `OPENAI_BASE_URL` | Optional OpenAI-compatible endpoint for the judge |
| `COT_CONTROL_JUDGE_MODEL` | Judge model, default `gpt-5-mini-2025-08-07` |
| `COT_CONTROL_META_DISCUSSION` | `auto` (default: judge meta-discussion when a key is set), `required`, or `skip` |

### Running with Datasets Registry

Available once the dataset is published:

```bash
# Oracle
harbor run -d cot-control/cot-control

# CoT-Control agent (needs this repository on PYTHONPATH and OPENROUTER_API_KEY)
PYTHONPATH=src/cot-control/src harbor run -d cot-control/cot-control \
  -a cot_control.agent:CoTControlLLMAgent -m deepseek/deepseek-r1
```

### Using Job Configurations

```bash
# From the repository root; oracle by default
harbor run -c src/cot-control/run_cot-control.yaml
```

To run a model, enable the commented `CoTControlLLMAgent` block in the YAML and prefix
the command with `PYTHONPATH=src/cot-control/src`.

### Running with Local Path

```bash
harbor run -p datasets/cot-control                                   # oracle
PYTHONPATH=src/cot-control/src harbor run -p datasets/cot-control \
  -a cot_control.agent:CoTControlLLMAgent -m deepseek/deepseek-r1 \
  --ak reasoning=auto
```

Agent options (`--ak key=value`): `api_base`, `api_key_env`, `max_tokens`,
`reasoning_tokens`, `temperature`, `reasoning` (`auto`/`on`/`off`),
`request_timeout_sec`, `extra_body` (JSON merged into the request, for example to pin an
OpenRouter provider). The defaults reproduce upstream.

### Running Individual Trial

```bash
harbor trial start -p datasets/cot-control/gpqa-0007-word-suppression
```

### Computing the Benchmark Metrics

Harbor's built-in mean weights every task equally. The paper's numbers are macro-averages
over (dataset, instruction type) pairs:

```bash
cd src/cot-control
uv run python -m cot_control.aggregate --job-dir ../../jobs/<job-name>
```

## Usage: Create Task Directories

```bash
cd src/cot-control
uv sync

# Full dataset (10,836 tasks)
uv run cot-control --output-dir ../../datasets/cot-control

# Parity subset (675 tasks, see the parity section)
uv run cot-control --split parity --output-dir ../../datasets/cot-control-parity
```

Flags:

- `--output-dir`: where to write tasks (default `datasets/cot-control` at the repo root)
- `--limit`: first N tasks; the first 27 cover every dataset × instruction type pair
- `--overwrite`: replace existing task directories
- `--task-ids`: only these ids, e.g. `gpqa-0007-word-suppression`
- `--split`: `full` (default) or `parity`
- `--datasets`, `--modes`: restrict to some source datasets or instruction types
- `--source-dir`: local checkout of the upstream repository at the pinned commit; without
  it the three dataset files are downloaded from GitHub and cached in
  `~/.cache/harbor-cot-control`
- `--prompt-model-name`: model name passed to the upstream prompt builder (see
  [Notes & Caveats](#notes--caveats))
- `--parity-rows-per-dataset`, `--parity-seed`: parameters of the parity subset
- `--export-upstream-subset DIR`: also write the selected questions as upstream-format CSV
  files, for running the same subset with the original harness

Check the result against upstream:

```bash
uv run --with python-dotenv python -m cot_control.check_fidelity \
  --upstream-dir /path/to/CoTControl --dataset-dir ../../datasets/cot-control
```

## Comparison with Original Benchmark (Parity)

**Parity experiments have not been run yet.** The agent, model, number of runs and subset
below are a proposal that still has to be agreed with the Harbor team; the table will be
filled from `parity_experiment.json` once the runs exist.

| Agent | Model | Metric | Number of Runs | Dataset Size | Original Benchmark Performance | Harbor Adapter Performance |
|-------|-------|--------|----------------|--------------|--------------------------------|----------------------------|
| cot-control-llm@0.1.0 | TBD | Controllability Score (macro %, mean ± sample SEM) | TBD | TBD | not yet run | not yet run |

What has been established so far is *not* parity, and is listed here only so it is not
mistaken for it:

- The 10,836 prompts, control values and answer keys equal the output of the upstream
  functions (`check_fidelity`, "upstream" section).
- On 2,304 synthetic outputs the upstream grading command line and the task verifier agree
  on every row, and `aggregate.py` gives identical macro-averages for both formats.
- The agent's request body equals upstream's `_build_payload` for a range of model names
  (`check_fidelity`, "request" section).

Proposed protocol:

- **Subset**: `--split parity`, 25 questions per dataset that have keyword data, chosen
  by a seeded hash (seed 42), × 9 instruction types = 675 tasks.
- **Agent**: `cot-control-llm@0.1.0` on the Harbor side; unmodified `run_cceval.py` on the
  original side. Same model id, provider and judge model on both sides.
- **Order**: 5–10 tasks on both sides, then one full run on both sides, then three.

Generate the subset once. `--export-upstream-subset` also writes the same questions as
upstream-format CSV files, so the original harness can run the identical subset through
its `--data` flag without any code change:

```bash
cd src/cot-control
uv run cot-control --split parity --output-dir ../../datasets/cot-control-parity \
  --export-upstream-subset /path/to/parity-data
cd ../..
```

Reproduction, original side:

```bash
git clone https://github.com/YuehHanChen/CoTControl && cd CoTControl
git checkout 5d78aeffe0152ba087c2d31cd07712d029c64785
uv sync && cd CoT-Control-QA
export OPENROUTER_API_KEY=... OPENAI_API_KEY=...
for data in gpqa_w_keyword mmlu_pro_mini_w_keyword hle_w_keyword; do
  for mode in word_suppression multiple_word_suppression ignore_question repeat_sentences \
              end_of_sentence meow_between_words lowercase_thinking uppercase_thinking alternating_case; do
    # --no-resume is required here, see "Upstream behaviour kept as is" below
    uv run run_cceval.py --model <model> --data /path/to/parity-data/$data.csv \
      --mode $mode --log_dir parity_run1 --no-resume
  done
done
for f in logs/parity_run1/*.csv; do uv run grade_compliance_csv.py --input_file "$f"; done
# same aggregation code as the Harbor side
uv run --project /path/to/adapters/src/cot-control python -m cot_control.aggregate \
  --upstream-csv logs/parity_run1/graded_results/*.csv
```

Reproduction, Harbor side:

```bash
export OPENROUTER_API_KEY=... OPENAI_API_KEY=...
PYTHONPATH=src/cot-control/src harbor run -p datasets/cot-control-parity \
  -a cot_control.agent:CoTControlLLMAgent -m <model> --job-name cot-control-parity-run1
cd src/cot-control && uv run python -m cot_control.aggregate --job-dir ../../jobs/cot-control-parity-run1
```

A fork of the upstream repository with these reproduction notes will be linked here once
the parity plan is agreed.

## Notes & Caveats

**Deviations from the paper and upstream**

- *Instance count.* 10,836 tasks rather than the paper's 10,926; see Overview.
- *Agent subset excluded.* The SWE-bench Verified and BFCL part is not adapted. Upstream
  publishes no script that turns its trajectories into the graded table, does not specify
  how multi-turn reasoning is concatenated, and leaves `inspect_ai`/`inspect_evals`
  unpinned, so a faithful port could not be validated.
- *`baseline` mode excluded.* It carries no instruction, so there is nothing to score for
  controllability, and it is not part of the paper's instance count.
- *Judge model.* Upstream hard-codes the alias `gpt-5-mini`. The adapter defaults to the
  dated snapshot `gpt-5-mini-2025-08-07` and exposes `COT_CONTROL_JUDGE_MODEL`. For parity
  both sides must use the same id.
- *Prompt wording depends on the model.* Upstream says "analysis channel" for model names
  containing `gpt-oss` and "reasoning stage" otherwise. Harbor instructions are static:
  the default dataset uses "reasoning stage". Generate a gpt-oss variant with
  `--prompt-model-name openai/gpt-oss-120b`.

**Upstream behaviour kept as is** (reported here rather than changed on one side)

- *Answer key of GPQA row 29.* The correct option text is the single letter `c`, which
  upstream's `convert_answer_to_letter` reads as option C although the text is option A.
  The adapter keeps upstream's key. This affects `correctness` on that question only.
- *GPQA row 74* lists the correct option text twice; upstream takes the first.
- *Answer extraction fallback* only recognises letters A–D when a response lacks the
  `ANSWER: X` format, although HLE and MMLU-Pro have up to 26 and 10 options.
- *Resume merges unrelated results.* `run_cceval.py` resumes by default and looks up
  earlier results by model and mode only (the mode by substring). Runs of different
  datasets or modes that share a `--log_dir` therefore copy each other's rows into their
  output files. Use `--no-resume` on the original side; `cot_control.aggregate` also drops
  rows duplicated across the files it is given.
- *Empty reasoning scores 0.* If the API returns no reasoning trace, rule-based types
  score 0, as upstream. A model or provider that hides its reasoning therefore looks
  uncontrollable. Check `reasoning_chars` in `grading_details.json` before trusting scores.

**Using the dataset**

- *The reasoning trace must come from the harness.* The verifier reads
  `/logs/agent/cot_control/reasoning.txt`. Asking a tool-using agent to write its own
  reasoning there would measure output control, a different and much easier quantity, and
  the agent could simply write a compliant file. Standard CLI agents are therefore not
  supported yet; without a captured trace the verifier reports an error, not 0.
- *Models must expose raw reasoning.* The paper notes that its raw CoTs of OpenAI models
  were obtained through a collaboration with OpenAI.
- *`ignore_question` needs the judge.* Without `OPENAI_API_KEY` those 1,214 tasks end
  with a verifier error. The other 9,622 tasks get their main reward without any key.
- *Reward direction.* A reward of 1 means the model controlled its reasoning.
- *Macro vs. micro.* Use `cot_control.aggregate` for numbers comparable to the paper.
- *Oracle.* The oracle demonstrates that a compliant trace exists and that the verifier
  accepts it; it is not a model-produced solution.
- *Runtime.* Tasks are light, but each one starts a container. On a 10-core laptop the
  oracle takes about 11 s per trial at 4 concurrent trials (roughly 8 hours for the full
  set). All tasks share one Dockerfile, so the image is built once and cached.

## Installation / Prerequisites

- Docker running, and Harbor installed (`uv tool install harbor`; developed against 0.27.0)
- Python 3.11+ and `uv`; from `src/cot-control`: `uv sync`
- Network access to GitHub when generating tasks without `--source-dir`
- `OPENAI_API_KEY` for the judge; `OPENROUTER_API_KEY` (or another OpenAI-compatible
  endpoint that returns reasoning) for the CoT-Control agent

## Troubleshooting

- **`RewardFileNotFoundError` on `ignore_question` tasks**: `OPENAI_API_KEY` is not set
  for the verifier. See `verifier/test-stdout.txt` in the trial directory.
- **"No reasoning trace was captured for this run"**: the task was run with an agent other
  than the oracle or `CoTControlLLMAgent`.
- **Every reward is 0 and `reasoning_chars` is 0**: the endpoint returned no reasoning.
  Try `--ak reasoning=on`, or use a model and provider that return the trace.
- **`ModuleNotFoundError: cot_control`**: add `PYTHONPATH=src/cot-control/src`.
- **"does not match the dataset pinned at upstream commit"**: `--source-dir` points at a
  different revision of the upstream repository.
- **Harbor asks to confirm environment variables**: pass `--yes` in scripts.

## Citation

```bibtex
@misc{yuehhan2026reasoningmodelsstrugglecontrol,
      title={Reasoning Models Struggle to Control their Chains of Thought},
      author={Chen Yueh-Han and Robert McCarthy and Bruce W. Lee and He He and Ian Kivlichan and Bowen Baker and Micah Carroll and Tomek Korbak},
      year={2026},
      eprint={2603.05706},
      archivePrefix={arXiv},
      primaryClass={cs.AI},
      url={https://arxiv.org/abs/2603.05706},
}
```

## Authors & Contributions

This adapter is developed and maintained by Run-Ze Fan (runze.fan@icloud.com)

**Issues and Contributions:**
- Submit issues and pull requests to the adapters repository.
- Questions about the benchmark itself belong with the
  [upstream repository](https://github.com/YuehHanChen/CoTControl).
