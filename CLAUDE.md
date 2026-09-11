# ROTATE — Project Constitution

This file is read at the start of every session. It governs all work in this repo.
Companion documents:
- `swing-app-build-scope.md` — the WHAT: all rules, thresholds, formulas, data model
- `rotate-module-build-plan.md` — the HOW/WHEN: build order and acceptance gates

---

## Absolute rules

1. **ALL trading logic lives in `rotate/rules/`.** Never duplicate logic into `scan/` or
   `backtest/`. Both call the same functions. If they could ever disagree, the design is wrong.

2. **Every threshold lives in `config/thresholds.yaml`.** No magic numbers in code, ever.
   Percentiles, z-scores, day counts, size caps, correlation limits — all of it, config only.

3. **This app NEVER places trades.** No broker APIs. No order execution. No auto-trading of
   any kind. Output files only. The human places every order manually.

4. **Never use fundamental data with `report_date` > the simulation date.** Look-ahead bias
   is the #1 failure mode of this project. Where point-in-time data is unavailable, apply a
   conservative 45-day reporting lag and label those results as approximate.

5. **Prices must be split/dividend adjusted.** Verify against a known recent split before
   trusting any price series. An unadjusted 4-for-1 split looks like a 75% crash and the
   dip scanner will flag it every time.

6. **No base rate may be displayed with n < 30.** Print "insufficient data" instead.
   Always display n alongside every percentage. Never display a single probability for a
   specific trade.

7. **Base rates are computed over the broad universe, never the approved pool.** The approved
   pool is 40–60 companies already known to have survived; rates computed on it are fictional.

8. **The app never adds a ticker to `approved_pool` without explicit human approval.**
   The screener writes a review queue. A human promotes from it. Removal is never automatic.

---

## Scope discipline

- **Build only the module I name.** Do not scaffold, stub, or pre-create future modules.
- Do not add features not in `swing-app-build-scope.md`. Ask instead.
- Prefer boring, readable code. This is a **1-user app** with ~900 tickers and ~10 trades a
  year. Do not optimize for scale that will never arrive. No microservices, no queues,
  no caching layers, no abstractions "for later."
- Write tests first, then the implementation.
- When a module's acceptance gate passes, **stop and report**. Do not continue to the next one.

---

## Core philosophy (preserve through every design decision)

> Fundamentals decide **what**. Price action decides **when**.
> Portfolio risk decides **how much**. The human decides **yes or no**.

Fundamentals and price data must not contaminate each other's gates.

---

## Stack

- Python 3.11+
- pandas, numpy, scipy, statsmodels
- SQLAlchemy (data access), pydantic (config/schema validation), pytest (tests)
- **SQLite locally. Azure SQL only at Module 11.**
- No Docker, no cloud, no CI complexity until Phase 3 / Module 11.

## Repo layout

```
rotate/
  ingest/      price + fundamental fetchers -> normalized storage
  rules/       ALL decision logic (shared by scan + backtest)
  scan/        nightly job -> trade cards
  screen/      quarterly universe screen -> review queue
  portfolio/   positions, factor exposure, correlation, drawdown budget
  backtest/    replays rules/ over history
  baserates/   historical outcome distributions
  report/      nightly text file + dashboard views
  store/       database models and access layer
config/thresholds.yaml
tests/
data/          <- gitignored
```

---

## Testing

- pytest. Every module needs tests **before** it is considered done.
- Financial calculations need at least one **hand-verified fixture** — a number worked out
  on paper, asserted exactly.
- **Critical test:** `scan` and `backtest` must produce byte-identical signals for the same
  historical date. This is the most important test in the repo.
- Backtest fills: fill at `min(limit_price, day_open)`, only if the day's low traded through
  the limit. Subtract 5 bps slippage plus commission per side.

---

## Module order (build strictly in sequence)

| # | Module | Gate summary |
|---|---|---|
| 0 | Falsification baseline | 20yr crude backtest — record CAGR/DD/win rate as the control |
| 1 | `store/` | Tables create; insert/read round-trip |
| 2 | `ingest/prices` | Split-adjusted, verified by eye on a known split; idempotent re-run |
| 3 | `ingest/fundamentals` | `as_of` query returns nothing filed after that date |
| 4 | `rules/quality.py` | Top/bottom 20 of S&P 500 read sanely by eye |
| 5 | `rules/valuation.py` | Nov-2022 point-in-time run flags META, rejects earnings-collapse names |
| 6 | `rules/confirmation.py` | Signals plotted on charts; never fires in a straight-line decline |
| 7 | `rules/sizing.py` + `exits.py` | Hand-computed fixtures; floor and correlation blocks assert |
| 8 | `scan/` + `report/` | 20 random historical dates → mostly NO candidates |
| 9 | `backtest/` | Identical-signal test passes; beats Module 0 baseline; ablations run |
| 10 | `baserates/` + `screen/` | n<30 suppression; review queue requires approval |
| 11 | `portfolio/` risk + Azure | Stress numbers sane; 5 unattended nightly runs with alerting |

**Two legitimate stop points:** if Module 0 shows no edge, or Module 9 cannot beat Module 0,
stop and redesign. Reaching either conclusion cheaply is a successful outcome.

---

## Session protocol

Each session covers **one module**. Start fresh; do not carry context across modules.

Opening prompt:

> Read `CLAUDE.md` and `rotate-module-build-plan.md`.
> Build **only Module N: [name]**, per §[section] of `swing-app-build-scope.md`.
> Do not create, stub, or scaffold any other module.
> Write the tests first, then the implementation.
> When the acceptance test passes, stop and show me the test output.

Commit after every passing gate, with the module number in the message.
When things go wrong, revert to the last passing commit rather than patching confusion.

---

## Known limitations — never paper over these

- The approved pool is the real strategy and the app cannot validate it.
- The backtest is optimistic by construction: survivorship bias, point-in-time fundamental
  limits, today's index members applied to old history. Print a survivorship warning on
  every backtest report.
- 6–15 trades/year means a 20-trade sample cannot distinguish skill from luck.
- Four positions in beaten-down quality names will correlate in a crisis.
- No software can make the user honor a stop. The stop goes in at the broker on entry day.
