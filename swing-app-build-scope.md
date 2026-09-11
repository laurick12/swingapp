# Build Scope — Quality-Dislocation Swing Trading Assistant

**Document purpose:** Complete build specification. Defines goal, architecture, rules, data, hosting, and phased AI roadmap. See `CLAUDE.md` for non-negotiable constraints and `rotate-module-build-plan.md` for build order and acceptance gates.

**Working name:** ROTATE

---

## 1. Goal

Build a **decision-support tool** — not an auto-trader — that helps a single retail investor:

1. Identify high-quality companies that are temporarily beaten down but showing evidence of recovery.
2. Produce a complete, pre-decided trade plan (entry, stop, target, position size) before any money is committed.
3. Manage a concentrated portfolio of 3–5 positions with institutional-style risk controls.
4. Exit mechanically, then cycle back to the same or a new name ("rinse and repeat").

**Core philosophy — preserve through every design decision:**

> Fundamentals decide **what**. Price action decides **when**. Portfolio risk decides **how much**. The human decides **yes or no**.

**Explicit non-goals:**
- The app never places trades. It outputs plans; the user places orders at their broker.
- The app never predicts a price. It reports historical base rates with sample sizes attached.
- The app never adds a stock to the approved pool without human confirmation.

**Definition of success:** The user opens one file each morning. Most mornings it says "no candidates." When it does produce a candidate, every number needed to execute is on the card, and the user did not have to make a single judgment call under time pressure.

---

## 2. Users and scale

- **1 user.** Single-tenant. No multi-user auth, no roles, no teams.
- ~500–900 tickers in the scanning universe.
- 40–60 tickers in the approved pool.
- 3–5 concurrent open positions.
- 6–15 completed trades per year.
- Daily bars only. No intraday, no real-time streaming, no websockets.

This is a small-data application. Do not over-engineer for scale that will never arrive.

---

## 3. Vocabulary

| Term | Definition |
|---|---|
| **Universe** | Broad index membership (S&P 500, optionally + MidCap 400). App-managed, never edited by user. |
| **Approved pool** | 40–60 tickers the user has explicitly approved. Only these can generate trade cards. |
| **Review queue** | Quarterly output of suggested additions/removals. Requires human approval to act. |
| **Trade card** | The complete pre-decided trade plan for one candidate. |
| **Base rate** | Historical frequency of outcomes for statistically similar past setups. Never a prediction. |
| **Regime** | Macro classification: expansion / slowdown / contraction / recovery. |
| **Dislocation** | Price fell materially more than the underlying business fundamentals did. |

---

## 4. Architecture

### 4.1 Non-negotiable architectural rule

**All trading logic lives in `rotate/rules/`, called by both the live scanner and the backtester.** No logic may be duplicated. If the backtest and the live scan can ever disagree, the build is wrong. This is the most important structural requirement in this document.

```
rules/
  regime.py        classify_regime(macro_data, date) -> Regime
  quality.py       score_quality(fundamentals, industry_peers, date) -> 0-100 percentile
  valuation.py     score_dislocation(fundamentals, price_history, date) -> z-scores
  confirmation.py  check_turn(price_bars, date) -> bool + levels
  sizing.py        size_position(candidate, portfolio, regime) -> shares, pct
  exits.py         check_exits(position, bar) -> None | "stop" | "target" | "structure" | "time"
```

### 4.2 Component map

```
ingest/          data fetchers -> normalized storage
rules/           ALL decision logic (shared by scan + backtest)
scan/            nightly job: runs rules over approved pool -> trade cards
screen/          quarterly job: runs quality+valuation over full universe -> review queue
portfolio/       open positions, factor exposure, correlation, drawdown budget
backtest/        replays rules/ over history
baserates/       computes historical outcome distributions for analogous setups
report/          renders nightly file + dashboard views
store/           database access layer
```

### 4.3 Data model (minimum tables)

- `universe` — ticker, name, sector, industry, index membership, active flag
- `approved_pool` — ticker, date_approved, source (app_suggested | user_added), conviction_note, status (active | under_review | retired)
- `prices_daily` — ticker, date, OHLCV, **split/dividend adjusted**
- `fundamentals` — ticker, fiscal_period, **report_date** (critical — see §8), revenue, EBIT, FCF, net debt, EBITDA, shares_out, ROIC, margins
- `macro_daily` — date, series_id, value
- `regime_history` — date, regime_label, inputs_snapshot
- `signals` — date, ticker, state, all gate scores, generated card JSON
- `positions` — ticker, entry_date, entry_price, shares, stop, target, structural_low, state, day_count
- `trade_history` — closed trades, full lifecycle, exit reason, realized P&L, the base rate shown at entry
- `cooldown` — ticker, exit_date, cooldown_until

---

## 5. The nightly flow (Phase 1 deliverable)

Runs once per day after market close. Sequential gates. A name must pass every gate.

### Gate 0 — Regime classification (portfolio-level, runs first)

Inputs: 10Y–2Y Treasury spread, high-yield credit spread, ISM manufacturing PMI, unemployment rate trend (3m vs 12m), CPI direction.

Output: one label — `expansion`, `slowdown`, `contraction`, `recovery` — plus a **risk budget multiplier**:

| Regime | Risk multiplier | Factor tilt |
|---|---|---|
| Expansion | 1.0 | Value, smaller size acceptable |
| Recovery | 1.0 | Value, cyclical quality |
| Slowdown | 0.7 | Quality, low volatility |
| Contraction | 0.5 | Quality, low leverage only |

Start with a rule-based classifier (threshold logic on the five series). Phase 4 may replace it with a learned model.

### Gate 1 — Quality (is it a good company?)

Scored **within its own industry**, percentile-ranked:

- Return on invested capital — trailing 3-year average
- Operating margin — level and 3-year stability (low standard deviation scores higher)
- Free cash flow positive in ≥ 3 of last 4 fiscal years
- Net debt / EBITDA below industry median
- Revenue growth ≥ industry median over 3 years
- Market cap ≥ $10B; median 20-day dollar volume ≥ $50M

**Pass threshold:** composite score ≥ 75th percentile within industry.

### Gate 2 — Dislocation (is it actually cheap?)

Central question: **did the price fall further than the business did?**

Compute z-scores of current valuation vs. the company's **own 10-year history**:
- EV / EBIT
- Price / Free Cash Flow
- Price / Sales

Also compute each vs. current industry distribution.

**Pass threshold:** at least two of three metrics ≤ −1.0σ vs own history, AND trailing-twelve-month revenue and EBIT have not declined more than 15% year over year.

That second clause is the value-trap filter. A stock down 40% whose earnings are also down 40% is correctly priced, not cheap. Reject it.

Additional condition: price is **15–50% below its 52-week high**. Below 50% on a name passing Gates 1–2 is rare; flag for manual review rather than auto-passing.

### Gate 3 — Confirmation (has it stopped falling?)

Price data only. All three must be true:

1. Close is **above the 50-day moving average**, having been below it within the last 20 sessions
2. The 50-day MA **slope over the last 10 sessions is ≥ 0**
3. A **higher low** is confirmed — the most recent swing low is above the prior swing low (10-day fractal/pivot definition; document the exact definition chosen)

Record the confirmed higher-low price. It becomes the structural stop level.

### Gate 4 — Portfolio risk check (does this fit the book?)

Before emitting a card:

- **Correlation:** 120-day return correlation vs. each open position. If > 0.70 with any holding, reduce size by 40%. If > 0.85, suppress the candidate entirely.
- **Concentration:** maximum 2 positions per sector. Maximum 4 positions total.
- **Factor exposure:** decompose the resulting book across value / quality / momentum / size / low-volatility. Flag if any single factor exposure would exceed +1.5σ.
- **Drawdown budget:** portfolio down ≥10% YTD → sizes cut by one-third. Down ≥20% → no new entries at all.
- **Regime multiplier** applied to final size.
- **Cooldown:** ticker must not be within 20 trading days of its last exit.

### Position sizing

Base allocation: **fractional Kelly (one-half), capped at 22% of account per position, floored at 8%.** Below the floor, suppress the candidate — a position too small to matter isn't worth a slot.

Kelly inputs come from the base-rate engine (win rate and payoff ratio for matched historical analogues). Until the base-rate engine has n ≥ 30 for a setup type, fall back to flat 18% sizing and label it as such on the card.

Apply in order: `base_kelly × regime_multiplier × correlation_adjustment × drawdown_adjustment`, then clamp to [8%, 22%].

### Trade levels

- **Entry:** limit order at the close price of the confirmation day, GTC for 10 trading days. Cancel if unfilled.
- **Stop:** structural — just below the confirmed higher low. Not ATR-based. If price breaks that low, the setup that was bought is objectively void.
- **Target 1:** nearest prior consolidation zone above current price (volume-weighted price shelf on the decline). Sell **half** here.
- **Trail:** remaining half trails a stop below the 50-day MA, evaluated on **weekly closes only** (daily would shake you out repeatedly during a recovery).
- **Time stop:** 250 trading days. Exit if still below entry price.
- **Minimum reward:risk:** Target 1 must be ≥ 2× the stop distance. If not, suppress the candidate. A good company at a bad risk/reward is still a bad trade.

### Output — the nightly file

Plain text/markdown, written to disk and optionally emailed:

```
=== ROTATE — Tue 12 May 2026 ===

REGIME: Slowdown · tilt QUALITY + LOW VOL · risk multiplier 0.70
BOOK:   3 positions · 54% invested · YTD −4.2% (within budget)
FACTOR EXPOSURE: Value +1.4σ · Quality +0.9σ · Momentum −1.1σ · RateSens +1.6σ ← concentrated
STRESS:  2008 replay −31% · 2020 replay −22% · 2022 replay −18%

--- OPEN POSITIONS ---
ABC  day 34  +6.1%  HOLD  stop 318.00  T1 378.00  structure intact
DEF  day 51  −2.8%  HOLD  stop  37.10  T1  46.50  structure intact
GHI  day 12  +1.4%  HOLD  stop  88.20  T1 104.00  earnings in 4 days

--- CANDIDATE: JKL ---
Quality:     84th pct in industry · FCF+ 12/12 qtrs · NetDebt/EBITDA 1.2
Dislocation: EV/EBIT −1.9σ vs own 10yr · P/FCF −1.6σ · revenue TTM flat
             38% below 52-week high
Confirmation: reclaimed 50d ✓ · slope +0.4 ✓ · higher low 71.40 ✓
PLAN:        Entry 78.20 limit (GTC 10d) · Stop 70.10 (−10.4%) · T1 104.00 (+33%)
             R:R 3.2 : 1 · Size 14% ($14,000 / 179 sh)
             Half-Kelly 19%, reduced for 0.71 correlation to ABC
BASE RATE:   n=212 matched setups (slowdown regime, 20yr, broad universe)
             44% reached T1 · median winner +38% / 127 days
             38% stopped out · median loser −14%
             median drawdown after entry: −11%   ← expect this, it is normal
PORTFOLIO:   adds +0.3σ rate sensitivity — already your largest tilt

--- NO OTHER CANDIDATES ---
```

**Most nights this file will show no candidates. That is the system working correctly.**

---

## 6. The quarterly flow — where stock names come from

Two lists, two different doors.

**Universe (app-managed):** index constituents, refreshed automatically. The user never edits this.

**Approved pool (user-owned):** 40–60 names. Only these generate trade cards.

Once per quarter, the screener runs Gates 1 and 2 across the entire universe and writes a review file:

```
NEW CANDIDATES (4)   — quality pct, valuation z, one-line rationale
FALLING OUT (2)      — which quality criterion decayed and by how much
UNCHANGED (46)
```

Rules:
- The user approves or rejects each line. Nothing moves automatically.
- The user may **add any ticker manually** at any time. The app still scores it and displays a warning if it fails Gate 1, but does not block it. Flag as `user_added` so conviction-carried names are visible.
- **Removal is never automatic.** A decayed name is marked `under_review`, which blocks *new* entries but leaves existing positions under their original exit rules.

Rationale: the machine does the searching, the human does the approving. Full automation puts a value trap in the portfolio the first time a screen misreads a collapsing business. Full manual means only ever considering names the user happened to think of.

---

## 7. Backtesting requirements

Replay `rules/` over **20 years minimum** — the sample must include 2008, 2011, 2015–16, 2018, 2020, 2022.

**Required outputs:**
- CAGR, Sharpe, Sortino, max drawdown, longest drawdown duration
- Win rate, average win, average loss, payoff ratio, expectancy per trade
- Average and median holding period, by exit reason
- Distribution of post-entry drawdown (validates the "give it room" premise)
- How often the structural stop was hit on trades that subsequently recovered
- Benchmark vs. buy-and-hold SPY **and** vs. equal-weight buy-and-hold of the approved pool

**Ablation tests — run each gate in isolation.** For each of Gate 1, 2, 3, and 4, produce results with that gate disabled. If disabling a gate does not hurt performance, that gate is decoration and should be deleted.

**Parameter sensitivity:** test each threshold across a range, not a point. A parameter that works at 250 days but fails at 200 and 300 is overfit. Require a plateau, not a spike.

**Execution realism:**
- Fill at `min(limit_price, day_open)` — never assume the limit price when the day gapped through it
- Subtract 5 bps slippage plus commission per side
- A limit order only fills if the day's low traded through the limit

---

## 8. Data honesty requirements (read twice)

These are the failure modes that make a backtest look brilliant and a live account lose money.

**Split adjustment.** Use fully adjusted prices. An unadjusted 4-for-1 split looks like a 75% crash and a dip-scanner will flag it every time. Verify once against a known recent split before trusting anything.

**Point-in-time fundamentals.** Providers serve *current* financials. Backtesting "was this company profitable in Q3 2022" using data downloaded today is look-ahead bias. Store a **`report_date`** on every fundamental record and, in backtest, only use data where `report_date <= simulation_date`. Where point-in-time data is unavailable, apply a **conservative 45-day reporting lag** and explicitly label those results as approximate.

**Survivorship bias.** Screening today's index members over old history invisibly excludes every company that failed. Use point-in-time index constituent data if obtainable; if not, print a survivorship warning on every backtest report.

**Base-rate universe.** Compute base rates over the **broad universe**, not the approved pool. The approved pool is 40–60 companies already known to have survived — recovery rates computed on it will be fictional.

**Sample size discipline.** Never display a base rate with n < 30. Print "insufficient data" instead. Always show n alongside every percentage.

---

## 9. Tech stack and hosting (Azure)

**Language:** Python 3.11+.
**Libraries:** pandas, numpy, scipy, statsmodels, SQLAlchemy, pydantic, pytest.

**Local development first.** Build and validate the entire rule engine and backtester locally against SQLite. Do not deploy until the backtest runs end to end.

| Component | Azure service | Notes |
|---|---|---|
| Nightly scan, quarterly screen | **Azure Functions** (timer trigger) | Serverless; free grant covers this workload |
| Database | **Azure SQL Basic (DTU)** | Small dataset; never provision vCore General Purpose |
| Dashboard | **Azure App Service** (Linux B1, or F1 free) | Streamlit or FastAPI |
| Secrets | **Azure Key Vault** | Never commit keys |
| Reports, backtest artifacts | **Azure Blob Storage** | Hot tier, a few GB |
| Job-failure alerting | **Application Insights** | A silently failed nightly job is the likeliest real bug |
| CI/CD | **GitHub Actions** → Azure | pytest on every push |
| Phase 4 ML | **Azure Machine Learning** | Only when Phase 4 begins |

**Cost target:** ~$7–31/month for Phases 1–3. The dominant cost will be fundamental data, not Azure.

**Configuration:** every threshold lives in `config/thresholds.yaml`, not code. The backtester must load alternative configs for sensitivity testing.

---

## 10. Build phases

### Phase 0 — Falsification gate (1–2 days) — DO NOT SKIP

Write a deliberately crude backtest: buy any S&P 500 name trading 20%+ below its 52-week high that reclaims its 50-day MA; structural stop; 2R target; 250-day time stop. No quality gate, no valuation gate, no regime, no sizing logic.

Run over 20 years. **This is the control.** Every later phase must beat it.

### Phase 1 — Rule-based core (2–4 weeks)
Ingestion, `rules/`, nightly scan, approved-pool management, trade cards, position tracking, text report. SQLite, local.

### Phase 2 — Backtest + base rates (2–3 weeks)
20-year replay, all metrics in §7, ablation tests, parameter sensitivity, base-rate engine. **Stop here and evaluate honestly before going further.**

### Phase 3 — Risk engine + Azure deployment (2–3 weeks)
Factor decomposition, correlation-adjusted sizing, drawdown budget, stress replays (2008/2020/2022), regime classifier. Deploy. Dashboard.

**Then run live in alert-only mode for 2–3 months before committing real capital.**

### Phase 4 — AI layer (§11)
Only after Phases 0–3 are running and a live track record exists.

**Realistic total to a usable Phase 1–3 system: 6–10 weeks part-time.**

---

## 11. AI/ML roadmap (Phase 4+)

**Governing constraints — apply to every module below:**

1. **AI never overrides a rule-based gate.** It ranks, contextualizes, and flags. It does not veto or approve.
2. **Every AI output carries a confidence measure and the sample size behind it.**
3. **Every AI module must be independently ablatable** via config, to prove it adds value.
4. **No model trained on fewer than ~500 examples.** The user's own trade history will never suffice. Train on broad-universe historical setups.
5. **Walk-forward validation only.** Train years 1–10, test 11–12, roll forward. Random splits on time-series data produce look-ahead leakage and beautiful, meaningless results.

### 4a — Analogue matching (build first)
Nearest-neighbour retrieval over historical setups, matched on quality score, dislocation z-scores, drawdown depth, sector, regime. Returns the outcome distribution of the k most similar past situations. Transparent, inspectable, no black box.

### 4b — Recovery probability model
Gradient-boosted classifier estimating P(reaches Target 1 before stop). Output feeds Kelly sizing, never the buy/sell decision. **Mandatory:** SHAP values on every prediction, displayed on the card. Recalibrate quarterly; track Brier score.

### 4c — Value-trap classifier
Trained on historical quality+dislocation setups that *never* recovered within 3 years. Features: accruals quality, margin trajectory, debt maturity wall, share count trend, competitive-position deterioration. Output is a **warning flag**, never an auto-reject.

### 4d — LLM research digest
Per candidate: a short structured brief — why the market is pessimistic, what would have to change, what management has said. **Retrieval-grounded with source links only; no numeric predictions; no buy/sell language; never feeds sizing or gating.** Log every brief alongside eventual outcome for later audit.

### 4e — Learned regime classifier
HMM or clustering over macro series, replacing Phase 3 threshold rules. Must beat the rule-based classifier out-of-sample before replacing it. Keep the rule version as fallback.

### 4f — Post-trade review agent
After each closed trade, compare entry rationale, base rate shown at entry, and actual outcome; write a journal entry. Aggregate quarterly into a pattern report. Likely the highest real-world ROI module, because the dominant failure mode is execution inconsistency, not signal quality.

### 4g — Calibration monitor (mandatory alongside any of 4b–4e)
Log the probability shown at entry for every trade. Every 20 trades, compare predicted vs. realized. If the model says 45% and reality is 25%, surface it prominently and revert to rule-based sizing until recalibrated. **Ship with the first probabilistic model, not after.**

### Explicitly out of scope
- Deep learning on raw price series
- Sentiment-driven entry signals
- Any model that outputs a price target
- Reinforcement learning for position management
- Auto-execution of any kind

---

## 12. Known limitations — display these permanently in the UI

- **The approved pool is the real strategy, and the app cannot validate it.** Perfect execution on a bad list loses money.
- **The backtest is optimistic by construction** — survivorship bias, point-in-time fundamental limitations, today's index members applied to old history.
- **6–15 trades per year means results accumulate slowly.** A 20-trade sample cannot distinguish skill from luck. Expect 3+ years before the track record means anything.
- **Concentration is real.** Four positions in beaten-down quality names will correlate, especially in a crisis.
- **Deep-value setups cluster in bear markets.** The strategy will feel worst exactly when it is planting its best trades.
- **No software can make the user honor a stop.** That's why the stop goes in at the broker on the day of entry.

---

## 13. Acceptance criteria

1. `rules/` is the single source of truth — a test asserts scan and backtest produce identical signals for the same historical date.
2. The 20-year backtest runs end to end and produces every metric in §7.
3. Ablation results exist for all four gates.
4. Phase 0's crude baseline is beaten out-of-sample.
5. A nightly Azure Function writes the report file reliably, with failure alerting.
6. The quarterly screen produces a review queue requiring explicit human approval.
7. Every threshold is in config, not code.
8. Base rates never display with n < 30, and always display n.
9. Point-in-time fundamental handling is implemented and unit-tested.
10. The app has never, at any point, placed a trade.
