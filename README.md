# ROTATE

A decision-support tool (not an auto-trader) for identifying high-quality, temporarily
dislocated companies showing evidence of recovery, and producing complete pre-decided
trade plans.

**Start here:**
- `CLAUDE.md` — project constitution, read at the start of every session
- `swing-app-build-scope.md` — full build specification (WHAT)
- `rotate-module-build-plan.md` — module-by-module build order and acceptance gates (HOW/WHEN)

## Golden rule

> One module. One test gate. No forward progress until the gate passes.

Build strictly in the module order defined in `CLAUDE.md`. Do not scaffold ahead.

## Core philosophy

> Fundamentals decide **what**. Price action decides **when**.
> Portfolio risk decides **how much**. The human decides **yes or no**.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
pip install pandas numpy scipy statsmodels SQLAlchemy pydantic pyyaml pytest
pytest
```

## Status

| Module | Status |
|---|---|
| 0 — Falsification baseline | not started |
| 1 — `store/` | not started |
| 2 — `ingest/prices` | not started |
| 3 — `ingest/fundamentals` | not started |
| 4 — `rules/quality.py` | not started |
| 5 — `rules/valuation.py` | not started |
| 6 — `rules/confirmation.py` | not started |
| 7 — `rules/sizing.py` + `exits.py` | not started |
| 8 — `scan/` + `report/` | not started |
| 9 — `backtest/` | not started |
| 10 — `baserates/` + `screen/` | not started |
| 11 — `portfolio/` risk + Azure | not started |

Update this table as each acceptance gate passes.
