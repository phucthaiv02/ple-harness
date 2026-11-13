# ple-harness

Entry for the Kaggle competition "Google - Agent Harness on Physical-Engineering's Last Exam" (PLE):
https://www.kaggle.com/competitions/physical-engineerings-last-exam

This repo is the submission: a WebSocket agent harness, `submission.yaml`, `LICENSE`, the final `artifacts/`,
and `writeup/` figures. It becomes **public** at competition close.

## Workspace

```
workspace/
  ple/                official repo (google-research/ple): run kit/eval from here; local.env lives here
  ple-harness/        this repo
  ple-lab-notebook/   private evidence log + writeup drafts (github.com/phucthaiv02/ple-lab-notebook)
```

## Skills in this repo

- `ple-harness-dev`: setup, kit/eval commands, branches and tags, fair-play rules.
- `ple-agent-design`: harness architecture, tools, prompts, verification, cost and time control.
- `ple-lab-notebook`: mirror of the canonical copy in `ple-lab-notebook`. Use it after every run. If you change it, update the canonical copy too.

The `ple-writeup` skill lives in the `ple-lab-notebook` repo.

## Rules for working here

- **Never commit:** keys, dev or hidden task files, run folders, or notebook content.
- **Branches and tags:** `main` holds the locked submission. Experiments go on `exp/<idea>` branches, and every evaluated commit gets a `run-<MMDD>-<letter>` tag.
- **Artifacts:** pass `--no-export` during experiments, and commit `artifacts/` only for the final run on `main`. The repo including history must stay ≤ 1 GiB.
- **Isolation:** the harness must never read graders, tests, reference files, `artifacts/`, run folders or git history.
- **Official pages win:** the Kaggle competition pages override repo docs. Re-check them each session.
