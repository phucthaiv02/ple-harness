---
name: ple-harness-dev
description: Use when building, running, debugging or iterating an agent harness for the Kaggle PLE (Physical-Engineering's Last Exam) competition: setup, kit check, eval/run.py, local models, repo conventions, fair-play rules.
---

# PLE harness development

Competition: https://www.kaggle.com/competitions/physical-engineerings-last-exam
Official repo: https://github.com/google-research/ple (start from `sample_submissions/`).
Reply to the user in the language they write in.

Companion skills:
- `ple-agent-design`: architecture and code of the harness.
- `ple-lab-notebook`: log every run.
- `ple-writeup`: the final writeup. Lives in the `ple-lab-notebook` repo.

## Workspace layout (keep the three repos side by side)

```
workspace/
  ple/                the official repo clone: run kit/eval from here; holds local.env (gitignored). Never commit here.
  ple-harness/        github.com/phucthaiv02/ple-harness: the entry (public at competition close)
  ple-lab-notebook/   github.com/phucthaiv02/ple-lab-notebook: private evidence log + writeup drafting
```

Conventions for `ple-harness`:

- **Branches:**
  - `main` holds the locked submission: harness, `submission.yaml`, `LICENSE` (Apache-2.0 or MIT), the final `artifacts/`, and `writeup/` figures.
  - `exp/<idea>` branches hold experiments. Merge a branch into `main` only when its ablation is logged as kept.
- **Tags:** tag every commit you evaluate as `run-<MMDD>-<letter>` so each notebook row maps to a SHA.
- **Repo size:** the repository including `.git` history must stay under 1 GiB, and `artifacts/` can be up to 512 MiB. So always pass `--no-export` during experiments, and commit `artifacts/` only for the final run on `main`. If it was committed several times, squash the history before submitting.
- **Never put any of these in this repo:** notebook files, run folders, dev task files, or keys. The repo becomes public, and the build bundles every tracked file.

## 0. Source of truth (check first in every new session)

1. **Kaggle official pages win**: Overview, Evaluation, Rules, and host/Kaggle-staff replies in Discussion. Re-read them (Claude in Chrome or WebFetch) at the start of each work session and after any long break, because hosts update them (the Evaluation tab was already updated once).
2. Repo docs (`README.md`, `docs/rules.md`, `kit/README.md`, `eval/README.md`, `docs/protocol.md`, `docs/models.md`, `docs/local-models.md`) for commands and technical limits.
3. If the two disagree, follow Kaggle, tell the user about the conflict, and log it in `ple-lab-notebook/decisions.md`. Known conflict: repo `docs/rules.md` and the samples README say `artifacts/` must come from the declared `model_api`; the Kaggle Evaluation tab and a Kaggle staff reply say `artifacts/` may be generated with a local or personal-API model, provided the writeup states which model and judge produced them.
4. Never rely on memory for dates, limits or prizes; re-check. Known deadline at the time this skill was written: final submission 2027-01-05 23:59 UTC.

## 1. Setup (once)

```bash
uv tool install harbor==0.20.0 --with websockets
export HF_TOKEN=<read token; accept dataset terms on HF first>
cd workspace && git clone https://github.com/google-research/ple
# first time only: seed the entry from a sample
cp -r ple/sample_submissions/<sample>/. ple-harness/
```

- Pick a sample: `minimal` (single loop), `plan-verify` (planner, worker, verifier with time budgets), `custom-openai` (MCP tools, streaming, context compaction, deadline-aware retries), `local-model` (same harness on a self-served model), or the adapters `pi-opus`, `hermes-opus` and `agy-gemini`.
- Keep secrets in `ple/local.env` (gitignored), **never** in the entry repo or `submission.yaml` `env:`.
- No paid cloud is required for dev. Options:
  - **Personal API key, OpenAI-compatible:** `model_api: openai-compatible/<model>` with a public HTTPS `model_base_url`, and the key as `MODEL_API_KEY` in `local.env`. Example: DeepSeek V4.1 Flash, `openai-compatible/deepseek-flash`, `https://api.deepseek.com/v1`. Run DeepSeek outside peak hours: peak is 01:00–04:00 and 06:00–10:00 UTC on weekdays, when prices double.
  - **Native providers:** `openai/`, `anthropic/` or `gemini/<model>`, with the key in `local.env`.
  - **Self-served model:** Ollama, vLLM, SGLang or llama.cpp, with tool-call parsing enabled and the Caddy auth gate in front. Pass `--model-base-url http://localhost:8090/v1`. On Linux the server must listen on `0.0.0.0`. See `docs/local-models.md`.
  - **Sponsored models on own GCP:** `kit/model_proxy/start.sh` (billed to the user's own project). The default judge through the proxy is Claude Opus 5.5, so set `JUDGE_MODEL` to a cheap model to avoid surprise cost.
- **Before submitting,** switch `model_api` to one of the 8 `google-cloud/<id>` models. Test 2–3 tasks on it so the harness matches its API format.
- **Data rule:** the competition data is for competition use only. Do not send dev task files to an API whose terms allow training on inputs until a host confirms that is allowed (this question is open on the forum). Check each provider's data terms.

## 2. Inner loop (run from `ple/`)

```bash
uv run python -m kit check ../ple-harness --env-file local.env [--no-smoke] [--cancel-test]
uv run python eval/run.py ../ple-harness --env-file local.env --no-export --tasks <id,id>   # subset while iterating
uv run python eval/run.py ../ple-harness --env-file local.env --no-export                    # full 29 tasks
uv run python eval/run.py --regrade runs/<run> --env-file local.env                          # judge failed only
uv run python -m kit up ../ple-harness   # then: eval/run.py --url ws://127.0.0.1:<port> --name X --tasks T
```

- **Judge:** set `JUDGE_BASE_URL`, `JUDGE_API_KEY` and `JUDGE_MODEL` to a cheap or local OpenAI-compatible model. With an `openai-compatible/` entry and no judge settings, the entry's own model is used as judge. Use the `checks` tasks (no judge needed) first for fast, reliable signal.
- **Fast feedback:** iterate on the 12 programmatic `checks` tasks before spending on the rubric tasks. The 12 rubric tasks with MCP world images take about 17 GB of images and long builds.
- **Commit and tag before each logged run** (or record the dirty diff) so every score maps to a 40-character SHA.
- **After EVERY run, follow the `ple-lab-notebook` skill.** Log the run, failures and the ablation before changing anything else. A run that is not logged cannot be used in the writeup.

## 3. Runtime constraints to design for

- **Serve container:** 2 CPU, 4 GB RAM, read-only root, 1 GB `/tmp` and 1 GB `$HOME`. The WebSocket must be accepted on `0.0.0.0:$PORT` within 120 s.
- **Time limit:** 6 h per task. Always send `done` or `error`; a timeout without one scores 0.
- **Egress:** only the declared model's host is reachable. No web search, no provider-hosted tools (they return HTTP 400), and no other APIs. An HTTP 400 counts against the score.
- **Task protocol:** `exec`, `read_file`, `write_file`, and `http` (to the task's MCP servers). At most 8 concurrent requests. Each `exec` runs in a fresh subshell. Output is truncated at 100 KB. Reads and writes are capped at 10 MB each.
- **Grading:** only the declared deliverable paths (`eval/dev_set.yaml`) and, for some tasks, `final_message.md` are graded. Write exactly the paths, file names and formats asked for.
- **Constraint and objective tasks** score `s = 0.4p + 0.6*G*O'`: breaking any hard constraint caps the item at 0.4. Verify hard constraints before optimizing cost.
- **Install:** everything goes under `/opt/harness` (`uv venv --python 3.12 /opt/harness/venv`). The build gets 30 min, and the bundle is capped at 2 GiB compressed. `apt-get` packages do not carry over to the serve container.
- **Claude via OpenAI SDK:** mark cache breakpoints (see `docs/models.md`), or every turn re-pays the full context.
- **Stage 2 ranking:** mean score over 3 runs × 56 hidden tasks, ranked on the score-cost Pareto front, with a minimum of 0.138. Track tokens and cost from day one.

## 4. Fair play (hard rules: refuse to break them, and warn the user)

- Never let the harness read graders, `tests/`, reference solutions, `artifacts/`, run folders or git history. Never branch on a task ID or hard-code answers, dev tasks included. The organizers re-run a random dev sample live, and hard-coding gets an entry disqualified.
- While developing, do not paste verifier code or reference outputs into the agent's prompts. Scores obtained that way are invalid. A public forum study admitted exactly this mistake.
- Never hand-edit `artifacts/`. Only `eval/run.py` or `kit artifacts --from <run>` may write it.
- Never store, log or share hidden test task content.
- No private code sharing outside the team; public forum or notebook sharing only.
- Make sure `model_api` matches the model the harness actually calls at submission.
