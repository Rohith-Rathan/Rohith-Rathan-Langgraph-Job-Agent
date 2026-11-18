# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A LangGraph **workflow** (not a multi-agent system) that ingests a messy job description: extract fields with Claude, ask a human for anything missing, reject duplicates, score/rewrite for quality, then publish to MongoDB. Nodes are fixed steps; only extraction, scoring and rewriting make LLM calls. See `README.md` for full setup and the graph diagram.

## Commands

```bash
pip install -r requirements.txt          # use a venv; Python 3.10+ (3.11–3.13 safest)
cp .env.example .env                     # needs ANTHROPIC_API_KEY and MONGO_URI

python app.py                            # Flask chat UI at http://127.0.0.1:5000
python jd_intake_pipeline.py "<raw JD>"  # CLI; prompts in-terminal for missing fields
python jd_intake_pipeline.py --resume <thread_id> "<answer>"   # resume a killed, paused run
```

There is no test suite, linter, or build step. Run commands from the project folder: `.env` and `checkpoints.sqlite` are resolved relative to the current directory.

## Architecture

- `jd_intake_pipeline.py` — the whole graph: `JDState` TypedDict, node functions, `route_*` functions, `build_graph(checkpointer)`, and the CLI runner. Module-level `llm` is a single `ChatAnthropic` (model from `ANTHROPIC_MODEL`, default `claude-sonnet-5-5`).
- `jobs_db.py` — MongoDB access (`jd_intake_db.jobs`), lazily creating the client from `MONGO_URI`. `find_duplicate_job` matches on stored `title_lower`/`company_lower` fields (deliberately not regex); `publish_job_in_db` writes those fields alongside the parsed ones.
- `app.py` — Flask wrapper over the same graph via `build_graph(SqliteSaver(...))`. `/chat` is stateless: a request with a `thread_id` is treated as the answer to a pending `interrupt()` (`Command(resume=...)`); without one it starts a new run. The frontend (`templates/index.html`, plain JS) holds the `thread_id` between turns.

### Conventions that matter when editing the graph

- **Routing is plain Python, never LLM-decided.** Duplicate rejection, `QUALITY_THRESHOLD` (75), `MAX_REVISIONS` (2) and `MAX_MISSING_INFO_ATTEMPTS` (6) are constants at the top of `jd_intake_pipeline.py` enforced in `route_*` functions. Keep new limits as code rules.
- **Human-in-the-loop via `interrupt()`** in `request_missing_info`, which asks for ONE field at a time and writes the reply verbatim into `parsed` (no re-parse). On resume the node re-runs from the top, so no side effects before `interrupt()`.
- **State persists in SQLite (`checkpoints.sqlite`, keyed by `thread_id`)**, not memory — this is what makes resume-after-kill work. Don't swap in `MemorySaver`.
- Nodes append to `state["log"]` via `_log()`; both the CLI and `/chat` print this as the audit trail. Terminal statuses: `published`, `rejected`, `needs_input` (the shared `escalate` node).
- LLM JSON replies are parsed defensively (`_strip_code_fence`, `extract_text` for list-of-blocks content); parse failures degrade to empty fields / score 0 rather than raising.
- `parse_jd`'s conditional-edge map includes an `escalate` target that `route_after_parse` never returns.

### Windows/LangSmith gotcha

`jd_intake_pipeline.py` force-disables LangSmith tracing at import (env overrides and popping API keys) because the tracer hangs on `platform.win32_ver()` on some Windows/Python 3.14 setups. `LANGSMITH_*` values in `.env` are intentionally ineffective unless those overrides are removed. It also calls `sys.stdout.reconfigure(encoding="utf-8")` at import.
