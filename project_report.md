# NIFTY Volatility Forecasting & Regime-Gated Trading — README

A complete pipeline that fixes a real pricing bug in an implied-volatility engine, builds a
volatility surface, tests unconditional forecasting (and finds it doesn't beat the market),
discovers a genuine directional edge on a specific 6–25% subset of trading days, and turns
that edge into a real, capital-tracked options backtest.

## How to Run This

1. Create one local folder (referenced as `FOLDER` / `folder` in every notebook — edit this
   path at the top of each notebook to point at your own directory).
2. Place all raw data files in that folder (see **Data Requirements** below).
3. Run the notebooks **in numeric order, 01 → 13**. Each notebook reads files produced by an
   earlier one — running out of order will fail with a missing-file error.
4. Every notebook prints its own progress and validation checks as it runs — read the printed
   output, not just the final cell, since several notebooks include mid-run sanity checks
   (e.g. "Sample balance check") that tell you whether to trust the final numbers.

## Data Requirements

| File pattern | Source | Used by |
|---|---|---|
| `OPTIDX_NIFTY_CE_*.csv`, `OPTIDX_NIFTY_PE_*.csv` | Raw NSE option chain (Open/High/Low/Close required) | 01, 02, 03, 04, 13 |
| `FUTIDX_NIFTY_*.csv` | Raw NSE futures | 01, 05, 07, 08, 09, 12, 13 |
| `hist_india_vix_*.csv` | NSE India VIX history | 06, 09, 12, 13 |
| `vix_us_daily.csv` | CBOE VIX (public source) | 08, 09, 12, 13 |
| `daily_volatility_features.csv` | Produced by notebook 03/04's pipeline | 05, 06, 07, 08, 09, 12, 13 |
| `smile_features.csv` | Produced by notebook 04 | 05, 07, 09 (where used) |
| `options_liquid.csv` | Produced by notebook 02 | 03, 04 |

**Note:** `sp500_daily.csv` was used in earlier notebook iterations but was **removed** from
the final feature set after testing showed it consistently *worsened* accuracy (see notebook
08 and the report, Section 5.2). It is no longer required by any notebook in this final set.

## Notebook-by-Notebook Guide

### `01_Data_Ingestion_and_Cleaning.ipynb`
Loads raw NSE files, parses dates, coerces numeric columns (handling `'-'` placeholder
values), and does basic structural cleaning. No modeling.

### `02_Expiry_and_Liquidity_Filter.ipynb`
Filters the raw option chain down to liquid, usable quotes (minimum volume, minimum open
interest, minimum premium, moneyness range, minimum days-to-expiry). Outputs
`options_liquid.csv` — **386,942 rows** in the verified run, down from ~2.1M raw quotes.

### `03_Implied_Volatility_Engine.ipynb`
**The core fix.** Diagnoses a systematic IV pricing bias (using raw spot + a flat assumed
interest rate instead of the market-implied forward), and fixes it by extracting the forward
price directly from put-call parity — no assumed interest rate needed. Re-solves implied
volatility for all liquid contracts using the corrected forward. Verified with a before/after
smile-shape check.

### `04_Volatility_Smile_and_Surface.ipynb`
Fits a smile curve (level, skew, curvature) to every (date, expiry) using a **regularized
smoothing spline** — the final result of three iterations (quadratic → quartic → spline),
each corrected after visual inspection caught a real shortcoming in the previous version.
Outputs `smile_features.csv` at 7D/14D/30D tenors.

### `05_Volatility_Prediction_Model.ipynb`
Predicts 30-day forward realized volatility. Compares Linear Regression and Gradient
Boosting, with and without smile-shape features. **Result: does not beat the base feature
set** — a genuine negative result, documented as such.

### `06_VIX_Prediction_Model.ipynb`
Predicts India VIX 5 days ahead. Compares naive persistence, Linear Regression, and two
Gradient Boosting variants (one overfit, one regularized). **Linear Regression wins.**

### `07_Volatility_Regime_Classification.ipynb`
Binary classification: will realized volatility enter a "spike" regime? Uses
`GradientBoostingClassifier`/`LogisticRegression` with class weighting and proper
time-series cross-validation. **Result: 0/30 recall on the one fold with real spike
examples** — the model is shown to be reactive, not predictive, and this is documented
directly rather than hidden.

### `08_NIFTY_IV_via_SP500_Spillover.ipynb`
Tests whether yesterday's U.S. S&P 500 / VIX move improves next-day NIFTY IV prediction.
**Every variant that adds skew or U.S. features underperforms the simplest NIFTY-only
baseline** — this result is why S&P 500 was dropped from the pipeline going forward.

### `09_Regime_Gated_Spillover.ipynb`
**The central notebook.** Builds the regime-gating logic step by step:
1. Flag days with an unusually large previous-day U.S. VIX move (75th percentile).
2. Within flagged days, check whether NIFTY's own price move *confirms* or *contradicts*
   what the U.S. signal implies (AGREE / DISAGREE / NEUTRAL).
3. Sweep the confirming-move-size threshold to find a strictness/accuracy tradeoff.
4. Build and test dedicated trained models for both the AGREE and DISAGREE subsets,
   comparing them against the simple fixed rule to see exactly where training adds value.

Nineteen cells, each with a printed explanation of what it tests and why. Read this notebook
top to bottom for the clearest walkthrough of the project's core idea.

### `10_Strictness_Sweep_and_Feature_Reference.ipynb`
Sweeps the strictness threshold continuously (not just at 2–3 fixed points) and produces the
full accuracy/coverage tradeoff curve, plus a complete feature reference table.

### `11_Feature_Correlation_and_Model_Explainer.ipynb`
Computes feature correlation with the target, broken out by regime (AGREE/DISAGREE/NEUTRAL),
and shows exactly which features drive the strict-subset accuracy improvement. This is where
the "several features flip sign between regimes" finding is documented and visualized.

### `12_Data_Driven_Agreement_Model.ipynb`
Tests whether a model given raw signals *and* engineered agreement/magnitude features (no
hand-coded bucket logic) can discover the regime-gating idea on its own. Also includes a
4-model comparison (Logistic Regression, Gradient Boosting, Random Forest, XGBoost) to
confirm Gradient Boosting is genuinely the best performer, not an arbitrary pick.

### `13_Rolling_6Month_PnL_Backtest.ipynb`
**The final, real trading result.** Retrains every strategy fresh every 6 months (no
lookahead, no reuse of future data), trades a calendar spread structure on the resulting
signal, and tracks one continuous, compounding Rs 100,000 portfolio per strategy across the
full 2021–2025 period. Produces the final equity curve (drawn as a true step function, with
a marker at every trade) and per-window P&L breakdown.

## Key Design Principles Used Throughout

- **No lookahead.** Every feature is explicitly lagged (`_prev` suffix); every trained model
  uses `TimeSeriesSplit` or genuine walk-forward retraining, never a random train/test split.
- **Every result is benchmarked.** No model's accuracy or P&L is reported without a naive
  baseline (persistence, majority class, or market-implied value) shown alongside it.
- **Negative results are kept, not hidden.** Notebooks 05, 06, 07, and 08 all report honest
  "this didn't work" findings, because they're informative about market efficiency.
- **Sample balance is checked before trusting P&L.** Notebook 13 (and its predecessors)
  print a LONG vs SHORT trade count — if this is heavily lopsided, it usually means uneven
  CE/PE data coverage, and the P&L numbers should not be trusted until that's fixed.

## Known Limitations (stated directly in the relevant notebooks)

- No transaction costs, brokerage, or slippage are modeled anywhere in the P&L backtest.
- The regime-gate thresholds (75th/25th percentile) are computed once from the full dataset,
  not recomputed per training window — a small, documented form of look-ahead in defining
  *which days are eligible to trade*, not in what is predicted on those days.
- The Manual Disagree strategy's per-window training set (5–10 rows) is often too thin to
  train reliably within the 6-month rolling window design — it produced no trades in the
  final backtest run for this reason.
