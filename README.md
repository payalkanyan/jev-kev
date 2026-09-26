# Confidence-Gated AI Support Desk

A small AI system that reads support tickets, makes fast **structured** decisions using a
small decision model (Jev or Kev), auto-routes high-confidence cases, and escalates
uncertain cases to a larger language model.

> **Research question**: Can a small decision model handle routine support tickets while
> preserving accuracy and reducing latency and inference cost?
>
> This is a **prototype** workflow, not a system to deploy on real customer accounts
> without additional privacy and safety checks.

## How it works

1. A ticket arrives, e.g. `"I was charged twice and my order hasn't arrived."`
2. A decision model (Jev API / Kev local / local rules baseline) predicts **category,
   urgency, sufficient-info, and escalation-need** — as structured fields, plus a
   confidence score.
3. The **confidence gate** auto-routes high-confidence predictions (team + template) and
   escalates low-confidence predictions to a larger LLM (or a mock human reviewer).
4. A dashboard surfaces predictions, confidence, escalations, latency, and cost.

## Architecture

```
src/support_desk/
├── schema.py              # canonical types: Category, Urgency, Decision, Ticket
├── generate_synthetic.py  # reproducible synthetic dataset generator (+ labels)
├── model/
│   ├── backend.py         # DecisionModel ABC — the contract every backend implements
│   ├── rules.py           # keyword heuristic baseline (always runnable)
│   ├── jev_client.py      # Jev HTTP decision model
│   ├── kev_client.py      # local Kev client (optional, [local] extra)
│   └── llm_fallback.py    # large LLM used to classify escalated tickets
├── pipeline/
│   └── triage.py          # the confidence gate: predict -> gate -> route/escalate
├── evaluation/
│   └── metrics.py         # accuracy, macro-F1, Brier score, ECE, latency, cost
├── dashboard.py           # Streamlit UI over persisted results + metrics
└── cli.py                 # `python -m support_desk {generate,pipeline,evaluate}`

tests/                    # pytest suite for models, triage gate, and metrics
data/                     # generated artifacts (git-ignored): tickets, results, metrics
```

**Design notes**
- The confidence gate (`pipeline/triage.py`) is the *single* escalation authority.
  Models only predict a `Decision`; the gate decides auto-route vs. escalate.
- Backends are pluggable: a new decision source only needs to implement
  `DecisionModel.predict(text) -> Decision`.
- The rules baseline lets the whole pipeline run end-to-end with no API keys.

See `agent.md` for the full project agent contract.

## Decision questions

- Which team should handle this ticket? → billing / delivery / technical / account
- Is the issue urgent? → low / medium / high
- Is there enough information to make a decision?
- Should the system escalate to a human?

## Three backends compared

| Backend | When it runs | Needs |
|---|---|---|
| `rules` | always (baseline) | nothing |
| `jev` | primary decision model | `JEV_API_URL` (+ optional key) |
| `kev` | local alternative to Jev | `local` extra + VRAM |
| `gpt-4o-mini` (LLM fallback) | only on escalated tickets | `OPENAI_API_KEY` (+ optional base URL) |

## Setup

Requires Python 3.11+. We use [`uv`](https://docs.astral.sh/uv/) (fast, no Poetry needed):

```bash
uv sync --extra dev          # creates .venv and installs the project + test deps
cp .env.example .env         # set JEV_API_URL / OPENAI_API_KEY as needed
```

> Non-`uv` users can `python -m venv .venv && pip install -r requirements.txt`.

## Run

```bash
# 1. Generate a labelled dataset (seeded, reproducible)
uv run python -m support_desk generate --n 1000 --seed 42

# 2. Run the confidence-gated pipeline
uv run python -m support_desk pipeline --model jev --threshold 0.85 \
    --fallback openai --llm-model gpt-4o-mini

# 3. Evaluate all measures
uv run python -m support_desk evaluate --results data/results_jev_0.85.jsonl

# 4. Browse results
uv run streamlit run src/support_desk/dashboard.py
```

Use `--model rules` for a no-keys smoke test.

## Evaluate

Metrics computed over the held-out result set (see `data/metrics_*.json`):

- Classification accuracy and macro-F1 (category + urgency)
- Confidence calibration: **Brier score** and **ECE** (expected calibration error)
- Percentage of tickets auto-routed vs. escalated
- Median and 95th-percentile latency
- Estimated cost per 1,000 tickets
- Comparison across backends (run pipeline once per `--model`)

## Three-day build plan

- **Day 1 — Understand & reproduce**: read the Jev overview and Kev README, test a few
  classification calls, generate the labelled dataset and run the rules baseline.
- **Day 2 — Build the decision system**: connect the Jev/Kev backend, implement structured
  predictions + confidence threshold, add the escalation route to the LLM fallback.
- **Day 3 — Evaluate & present**: run all backends on the same held-out split, compare
  accuracy / calibration / latency / cost, and build the Streamlit dashboard showing
  successes and failure cases.

## Resources

- [BuildWithJev](https://buildwithjev.com) — browse recent projects and demos.
- JEV-as-a-Judge research paper (arXiv, Sept 2026) — confidence-gated evaluation and escalation.
- Jev / Kev READMEs — model weights, evaluation methods, training instructions.

## Project files

| File | Purpose |
|---|---|
| `agent.md` | Project agent card: identity, capabilities, conventions. |
| `pyproject.toml` / `requirements.txt` | Dependencies and build config. |
| `.env.example` | Template for model API keys / endpoints. |
| `Makefile` | Convenience targets for the common commands. |
| `src/support_desk/`, `tests/` | Implementation and tests. |
