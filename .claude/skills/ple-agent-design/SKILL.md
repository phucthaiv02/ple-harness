---
name: ple-agent-design
description: Use when designing or coding the PLE agent harness itself: architecture, tools, prompts, verification loops, context/time/cost control, and model portability for engineering tasks.
---

# Designing and coding the PLE agent harness

Companion skills:
- `ple-harness-dev`: setup, run commands, repo conventions, and rules.
- `ple-lab-notebook`: log every experiment.
- `ple-writeup`: the final writeup.

Read the official repo docs (`ple/docs/protocol.md`, `ple/docs/models.md`, `ple/kit/README.md`) and the sample code before changing protocol-level behaviour. Reply to the user in the language they write in.

## 1. Mental model: where code runs

```
serve container (YOUR code)          evaluator          task container (task files, tools)
2 CPU, 4 GB, read-only root,   <--websocket-->   exec / read_file / write_file / http(MCP)
libs you installed in /opt/harness              no network; python3 + whatever libs the task image has
only egress: the declared model endpoint        deliverables must end up HERE at the declared paths
```

- **The model never touches files directly.** Every tool call becomes an `exec`, `read_file`, `write_file` or `http` request in the task container.
- **There are two places to run code:**
  - **(a) Task container,** via `exec`. Use only what the task image has, and probe it first.
  - **(b) Serve container.** Pull the bytes with `read_file`, process them with libraries installed under `/opt/harness`, and return text or images to the model. Use (b) for PDF rendering, DXF/IFC parsing and validation, and anything the task image lacks.
- **Size limits:**
  - A read is at most 10 MB, and `taskbridge.read_file` raises on bigger files. Split those first, for example `exec("split -b 9M f /tmp/part_")`, then read the parts.
  - `taskbridge.write_file` chunks large writes automatically.
  - `exec` output is cut to head and tail at 100 KB.
  - Each `exec` runs in a fresh subshell, so `cd` and `export` do not persist.
- **Only the declared model is reachable.** You cannot add a separate vision or helper model: other models get 403 in evaluation. A local model in the serve container must fit 2 CPU and 4 GB, which is too weak to be useful as a VLM.

## 2. Starting point

- Begin from `sample_submissions/custom-openai` (streaming, MCP discovery, compaction, retries, deadline handling) or `plan-verify` (role split). Keep `taskbridge.py` unchanged.
- Make every behaviour a switch read from `env:` in `submission.yaml` (`STAGES`, `MAX_FIX_ROUNDS`, `REASONING_EFFORT`, `COMPACT_AT_TOKENS`, `VISION`...). Each switch is an ablation, and a set of budget values gives the score-vs-cost curve. Log the active config at task start with `session.log` so the notebook can tie runs to variants.

## 3. Recommended pipeline (generic, task-agnostic)

Each stage is a separate function with its own time share. Deterministic harness code does what code can do; the model does judgment.

1. **Recon (code, then model).**
   - **Probe the environment:** `python3 -V`, `python3 -c 'import pkgutil;print(sorted(m.name for m in pkgutil.iter_modules()))'`, `which libreoffice pdftotext ...`.
   - **Inventory files:** `find` with sizes and types, grouped into drawings (PDF/DXF/DWG/IFC), specs, spreadsheets, email or docs, and data.
   - **List MCP tools:** `tools/list` when `mcp_servers` is non-empty.
   - Feed a compact summary to the model, not raw dumps.
2. **Deliverable contract (model, checked by code).** Extract from the instruction every required deliverable: exact path, format, required sections or fields, and any stated acceptance check. Store it as JSON in the harness and reuse it in later stages.
3. **Plan.** Method, inputs per step, the governing code or standard clauses to look up in the provided files, hard constraints, objective, and how each number will be verified.
4. **Execute.** Compute with scripts and save them next to the work. Write a skeleton of every deliverable early, then improve it, so a timeout still leaves something gradable.
5. **Verify (fresh context).** An independent verifier sees only the task, the contract, and the files. It recomputes key numbers, checks hard constraints, units, and each contract item, and returns PASS or a numbered list of gaps. Add **programmatic checks in code** before the model check:
   - each deliverable path exists, is a regular file, and is non-empty
   - `docx`/`xlsx` files open
   - IFC parses
   - JSON/CSV parse
   - `.py` deliverables run
6. **Fix rounds** (bounded), then **finalize.** Kill background processes, send a `final_message` that lists each file, its key results, assumptions, and the clauses used (rubric judges read it), then send `done`.

## 4. Tools to give the model

- **Core tools:** `bash`, `read_file`, `write_file`, `edit_file` (exact replace).
- **MCP:** expose the tools directly if the catalog is small; otherwise use `mcp_find` and `mcp_call` (catalogs can be about 500 KB).
- **Higher-value harness-side tools** (implemented in the serve container), each returning compact text:
  - `inventory()`: the file tree with types and sizes.
  - `pdf_text(path, pages)`: text with page numbers.
  - `view_page(path, page, dpi)`: renders a drawing page to PNG for vision models. Chat Completions tool results are text, so append the image as a following user message (`image_url` data URI). Check that the declared model accepts images first (send one test image), and fall back to text extraction if not.
  - `dxf_summary(path)`: layers, entity counts, text and dimension entities, extents (`ezdxf`).
  - `ifc_check(path)`: schema, element counts, GUID uniqueness, basic clash or overlap checks (`ifcopenshell`).
  - `xlsx_dump(path, sheet, range)`: values and formulas (`openpyxl`).
  - `check_deliverables()`: runs the programmatic checks from stage 5.
- **Drawings:** read structured data first (DXF entities, the PDF text layer, IFC, xlsx); it is exact and cheap. Use vision for markups, scans, layout sanity checks, and cross-checking what was extracted. Make vision a switch (`VISION=on/off`) and ablate it.
- **Licenses:** dependencies ship in the bundle under an Apache-2.0/MIT entry, so prefer permissive libraries (for example `pypdfium2` + `Pillow` for rendering rather than AGPL `PyMuPDF`). Check every license and keep `/opt/harness` under 2 GiB.

Example (serve-side page render):
```python
import base64, io, pypdfium2 as pdfium
async def view_page(session, path, page=0, dpi=110):
    pdf = pdfium.PdfDocument(await session.read_file(path))   # <=10 MB, else split first
    img = pdf[page].render(scale=dpi / 72).to_pil()
    buf = io.BytesIO(); img.save(buf, "PNG")
    return "data:image/png;base64," + base64.b64encode(buf.getvalue()).decode()
```

## 5. Prompts that encode engineering practice

- **Units ledger:** every quantity carries units. Convert explicitly in code and state the unit system of each source file.
- **Clause discipline:** cite the document, section or table, and edition used for every design value. Never use a value that is not in the project files or standard engineering knowledge; flag it as an assumption instead.
- **Hard constraints before the objective.** Many checks-tasks score `0.4p + 0.6*G*O'`, so one hard-constraint failure caps the item at 0.4.
- **Assumptions register and limitations section** in memos.
- **Memo skeleton for rubric tasks:** purpose; documents reviewed; method; calculations with inputs, equations and results; findings; recommendation or decision; open items.
- **Exact compliance with the requested deliverable:** path, file name, format, sheet names, columns.
- General engineering knowledge in prompts is fine. Anything derived from a specific dev task's grader, tests or reference solution is not.

## 6. Context, time and cost

- **Context:** clip tool outputs (head and tail). Compact at a token threshold: drop old tool outputs first, then summarize into notes. Keep a running `notes.md` in the task container's `/tmp` that the model updates, so a compacted agent can recover. `/tmp` is outside the deliverable paths and is not graded.
- **Time:** use `session.time_left()`. Give each stage a share of the time (plan about 10%, verify and fix about 25%). Cap every `exec` timeout before the wrap-up point. Start the wrap-up 3–4 minutes before the deadline. Handle `Cancelled` by sending the best `final_message`. Never exceed 8 concurrent requests.
- **Cost** (Stage 2 ranks on a score-cost Pareto front):
  - Report every call with `session.usage(...)`.
  - Use prompt caching: keep the conversation prefix stable and never rewrite old messages (Claude also needs cache marks).
  - Stop as soon as the verifier passes.
  - Make the effort level and number of rounds configurable.
  - Prefer cheap deterministic checks to extra model calls.
- **Streaming:** stream long calls, because the egress proxy kills connections idle for 30 minutes.

## 7. Robustness (each failure here costs a whole task)

- **Retry policy:** retry 408/409/429/5xx and dropped connections with backoff. On context overflow, compact instead of retrying blindly. Treat any other 400 as a bug: it counts against you in Stage 2.
- **Forbidden parameters and tools:** never send provider-hosted tools (search, code execution, URL fetch), which return 400.
- **Pass provider fields back:** return assistant messages as received (Gemini thought signatures, Claude signed thinking).
- **WebSocket hygiene:** `ping_interval=None`. Fewer than 20 malformed frames. Always send `done` or `error`.
- **Clean shutdown:** before `done`, stop background processes you started, including any started with `env -i`.
- **Wrap every tool in try/except** and return errors to the model as text instead of crashing the session.

## 8. Model portability (Cross-Model: 15 pts)

- Use the OpenAI-compatible endpoint (`OPENAI_BASE_URL`, `MODEL_NAME`) for every model. Gate model-specific details on `MODEL_NAME`: effort names, cache marks, max output tokens, context window, and vision support.
- Run the same commit on at least two models from different families, for example DeepSeek V4.1 Flash plus GLM 5.3 Flash, or one sponsored model. Record which harness changes help all models and which help only one.
- The final `model_api` must be one of the 8 `google-cloud/<id>` models, and the code must support its API format. Test a few tasks on it before submitting.

## 9. Development workflow

- **Unit-test harness-side tools offline** on local copies of dev task files (local competition use only; never commit task files to `ple-harness`).
- **Debug live** with `kit up` plus `eval/run.py --url ... --tasks <one task>`, or run `serve.py` on the host with a debugger.
- **Emit structured logs:** `session.log` lines such as `stage=verify verdict=FAIL gaps=3`, `tool=view_page n=12`, `contract_items=5`. This makes failure triage and the writeup figures cheap.
- **One change per experiment** on an `exp/<idea>` branch. Tag the commit, run the same task subset, and log it with `ple-lab-notebook`.

## 10. Fair play (refuse to violate)

- Never branch on a task ID, title or file fingerprint.
- Never hard-code answers.
- Never read graders, `tests/`, reference files, `artifacts/`, run folders or git history at build or run time.
- Never contact any host other than the model endpoint.
- Generic tools and prompts that would help on any engineering task are the goal.
