# ROTATE — Module-by-Module Build Plan

**Companion to:** `swing-app-build-scope.md` (the WHAT) and `CLAUDE.md` (the constraints)
**Primary tool:** Claude Code, run from repo root
**Editor:** Cursor or VS Code for reviewing diffs
**Optional second tool:** Codex for PR review and low-risk modules

---

## The rule that governs everything

> **One module. One test gate. No forward progress until the gate passes.**

Financial logic fails silently. A wrong z-score doesn't throw an exception — it quietly produces a bad trade card six months from now. Every module below has an acceptance test a human can eyeball. If you can't verify it by hand, it isn't done.

State this explicitly at the start of every session:

> "Build only module N. Do not scaffold, stub, or implement any other module. Stop when the acceptance test for module N passes and report."

Agents want to be helpful by building ahead. That is the thing to suppress.

---

## The 12 modules

### Module 0 — Falsification baseline
**Build:** A single crude script. Buy any S&P 500 name 20%+ below its 52-week high that reclaims its 50-day MA. Structural stop, 2R target, 250-day time stop. No quality gate, no valuation, no regime, no sizing logic.

**Gate:** Produces 20 years of trade-by-trade output with CAGR, max drawdown, win rate. **Record these numbers. They are the control every later phase must beat.**

**Why first:** One or two days. If this shows nothing, the whole idea needs rethinking before two months are spent. Cheapest possible way to be wrong.

---

### Module 1 — `store/` (schema + DB layer)
**Build:** SQLAlchemy models for all tables in §4.3 of the scope. Migrations. Nothing else.

**Gate:** Tables create cleanly; a test inserts and reads back one row from each.

---

### Module 2 — `ingest/prices`
**Build:** Daily OHLCV fetch → `prices_daily`. Adjusted. Incremental (don't refetch history daily). Handles missing days, halts, delistings.

**Gate — by hand; do not let the agent self-certify:**
1. Pick a stock that split in the last 3 years. Plot its price. **There must be no cliff.**
2. Cross-check 5 random closes against a public chart.
3. Re-run the fetch twice — no duplicate rows.

*This is the bug that silently wrecks everything downstream. Spend the extra hour.*

---

### Module 3 — `ingest/fundamentals`
**Build:** Quarterly financials → `fundamentals`, **with `report_date` on every row.**

**Gate:** A test asserts `get_fundamentals(ticker, as_of=2022-06-30)` returns nothing filed after that date. Write this test before the code.

*Hardest data problem in the project. If the provider doesn't give report dates, implement the 45-day lag fallback and label it loudly.*

---

### Module 4 — `rules/quality.py`
**Build:** Industry-relative quality percentile per §5 Gate 1.

**Gate:** Run on today's S&P 500. **Read the top 20 and bottom 20 by eye.** If recognizable high-quality compounders aren't near the top and obvious junk isn't near the bottom, the scoring is wrong. No test suite substitutes for this.

---

### Module 5 — `rules/valuation.py`
**Build:** Dislocation z-scores vs own 10-year history + industry, plus the earnings-decline value-trap filter.

**Gate — the one that matters most in the whole build:**
Run as of **November 2022** using point-in-time data. It should score **META as a dislocation** and should **reject** names whose earnings fell as far as their price. Hand-check 5 outputs against actual historical financials.

*If Module 5 is wrong, everything downstream is confidently wrong.*

---

### Module 6 — `rules/confirmation.py`
**Build:** 50-day reclaim, flat-or-rising slope, higher-low detection.

**Gate:** Run on 10 known historical recoveries. **Plot every signal on a chart and look at it.** Pivot detection is notoriously fiddly — it will look correct in code and be wrong on screen. Confirm it does *not* fire during a straight-line decline.

---

### Module 7 — `rules/sizing.py` + `exits.py`
**Build:** Fractional Kelly with caps/floors, correlation adjustment, regime multiplier, drawdown budget. Exit checks.

**Gate:** Hand-computed fixtures. Given entry 100, stop 90, account 100k → assert exact share count. Assert the 8% floor suppresses a candidate. Assert a 0.9-correlation candidate is blocked. Each is a one-line test with a number worked out on paper.

---

### Module 8 — `scan/` + `report/`
**Build:** Wire Gates 0–4 together over the approved pool. Emit the nightly text file.

**Gate:** Run for 20 random historical dates. Most should produce **no candidates**. If it produces candidates most nights, a gate is too loose — find which one before continuing.

---

### Module 9 — `backtest/`
**Build:** 20-year replay calling `rules/` — the identical functions the scanner calls.

**Gate — two parts:**
1. A test asserts scan and backtest produce **byte-identical signals** for the same historical date. The most important test in the repo.
2. Results beat Module 0's baseline. **If they don't, stop and redesign. Do not proceed to polish.**

Then run the ablations: disable each gate in turn. Any gate that doesn't hurt performance when removed is decoration — delete it.

---

### Module 10 — `baserates/` + `screen/`
**Build:** Historical analogue statistics over the broad universe (not the approved pool). Quarterly screener → review queue.

**Gate:** Base rates refuse to display below n=30. Screener output requires explicit approval to modify `approved_pool`.

---

### Module 11 — `portfolio/` risk engine + Azure
**Build:** Factor decomposition, correlation matrix, stress replays (2008/2020/2022), then deploy.

**Gate:** Stress test numbers are sane. Nightly Azure Function runs 5 consecutive days without intervention and alerts on failure.

---

## Where it lives, and when

**Modules 0–10: entirely local.** Python + SQLite on the laptop. No cloud, no Docker, no CI complexity. The backtest is the heaviest thing you'll run and it's minutes. Cloud infrastructure during the logic-building phase adds deployment friction to every iteration and buys nothing.

**GitHub from day one** — private, `data/` gitignored. Not for collaboration; for `git diff` and clean reverts. Commit after every passing gate with the module number in the message. That history becomes the rollback ladder.

**Azure at Module 11 only**, when the logic is proven and the only remaining question is "how does this run unattended at 6pm."

| Phase | Where | Why |
|---|---|---|
| Modules 0–10 | Local, SQLite, private GitHub | Fast iteration, zero deploy friction |
| Module 11 | Azure Functions + Azure SQL + App Service | Unattended scheduling is the only thing the cloud is needed for |
| Live alert-only, 2–3 months | Azure | Prove it runs; prove you can leave positions alone |
| Real capital | — | Only after the above |

---

## Session discipline with the agent

**One module per session.** Start fresh — don't let a 3-hour context accumulate across modules; that's when agents contradict earlier decisions.

**Opening prompt template:**

> Read `CLAUDE.md` and `rotate-module-build-plan.md`.
> Build **only Module N: [name]**, per §[section] of `swing-app-build-scope.md`.
> Do not create, stub, or scaffold any other module.
> Write the tests first, then the implementation.
> When the acceptance test passes, stop and show me the test output.

**After every module:** read the diff yourself. Not all of it — but read `rules/` line by line, always. That's where money is won and lost. Ingestion and reporting can be skimmed.

**When it goes wrong:** revert to the last passing commit rather than asking the agent to patch its own confusion. Patching compounds; reverting doesn't.

---

## Realistic timeline, part-time

| Modules | Time |
|---|---|
| 0 (baseline) | 1–2 days |
| 1–3 (data) | 1 week |
| 4–7 (rules) | 1.5–2 weeks |
| 8 (scan + report) | 3–4 days |
| 9 (backtest + ablations) | 1 week |
| 10–11 (base rates, risk, Azure) | 1.5 weeks |

**~6–8 weeks part-time to a deployed alert-only system.** Then 2–3 months of live observation before any capital.

The two gates that can legitimately end the project early: **Module 0** (no baseline edge) and **Module 9** (full system can't beat the baseline). Both are designed to fail fast and cheap. Reaching either conclusion is a successful outcome, not a wasted effort.
