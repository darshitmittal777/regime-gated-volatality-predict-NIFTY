# Project Audit Summary — Feature & Model Reference

## Complete Pipeline: What Each Notebook Predicts, With What Model, Using What Features

| # | Notebook | Predicts | Model(s) | Key Features Used |
|---|---|---|---|---|
| 01 | Data Ingestion | — | none (data cleaning) | — |
| 02 | Expiry/Liquidity Filter | — | none (data filtering) | — |
| 03 | Implied Volatility Engine | — | none (BSM solver + put-call parity forward extraction) | — |
| 04 | Volatility Smile & Surface | — | none (regularized smoothing spline fit) | — |
| 05 | Realized Vol Prediction | 30D forward realized vol | GradientBoostingRegressor, LinearRegression | ATM/PUT/CALL IV (30D,14D), SKEW, RV_trailing, VRP, Term_Slope, Smile_Level/Skew/Curv |
| 06 | VIX Prediction | India VIX, 5 days ahead | GradientBoostingRegressor, LinearRegression | VIX OHLC, IV features (same as NB05), Smile features |
| 07 | Vol Spike Classification | Binary: realized vol spike regime | GradientBoostingClassifier, LogisticRegression | Same feature set as NB05/06 |
| 08 | NIFTY IV via SP500 Spillover | NIFTY ATM_IV_30D level, next day | GradientBoostingRegressor, LinearRegression | ATM_IV_30D_prev, SKEW_30D/14D_prev, **US_SP500_prev_return**, US_VIX_prev_close/chg1d |
| 09 | Regime-Gated Spillover | Binary: NIFTY IV direction, next day | LogisticRegression (multiple variants across cells) | See below — **feature list changes across cells within this notebook** |
| 10 | Strictness Sweep | — | none (threshold sweep analysis on NB09 output) | — |
| 11 | Feature Correlation Explainer | — | none (correlation analysis) | — |
| 12 | Data-Driven Agreement Model | Binary: NIFTY IV direction, next day | GradientBoostingClassifier (best), LogisticRegression | US_VIX_prev_chg1d/close, NIFTY_prev_return, SKEW_30D/14D_prev, India_VIX_prev_chg1d, + 5 engineered agreement/magnitude features |
| 13 | Vega Structure Backtest (Calendar + Butterfly) | Same as NB09/12, applied to real P&L | LogisticRegression, GradientBoostingClassifier | Same as NB09's DIS_FEATS / NB12's engineered set |
| 14 | Yearly P&L Backtest (train once) | Same, tested year-by-year on fixed 2021-2022H1 training | LogisticRegression, GradientBoostingClassifier | Same as NB13 |
| 15 | Rolling 6-Month P&L Backtest | Same, retrained every 6 months | LogisticRegression, GradientBoostingClassifier | Same as NB13/15 |

**Notebook 13 (both options-based and futures-based P&L versions) has been removed** — the options version showed structurally negative P&L unrelated to signal quality (theta decay/premium dynamics dominate any correct IV-direction call on a same-day round trip), and the futures version tested price-direction, which is a fundamentally different target than what this project's models actually predict (IV direction). Notebook 13's calendar spread structure is the theoretically correct and empirically validated replacement for both.

## Notebook 9's Internal Feature-List Evolution (all in one notebook, at different stages)

Notebook 9 is the notebook where the project's core idea (US VIX regime gate + NIFTY confirmation) was developed iteratively. Its feature lists changed across cells as the idea was refined:

| Cell | Purpose | Features | Includes S&P? |
|---|---|---|---|
| Cell 4/5 | Early logistic regression on all flagged days | `US_SP500_prev_return`, `US_VIX_prev_close`, `US_VIX_prev_chg1d`, `SKEW_30D_prev`, `SKEW_14D_prev`, `India_VIX_prev_close`, `India_VIX_prev_chg1d` | **Yes** (superseded) |
| Cell 11 | Trained model with agree/disagree category as input | `US_VIX_prev_chg1d`, `US_VIX_prev_close`, `NIFTY_prev_return` + category dummies | No |
| Cell 16 | Dedicated disagree-day model | `NIFTY_prev_return`, `US_VIX_prev_close`, `US_VIX_prev_chg1d`, `SKEW_14D_prev`, `SKEW_30D_prev` | No |
| Cell 18 | Dedicated agree-day model | Same as Cell 16 | No |

**This is left as-is in notebook 9** (not retroactively edited) because it accurately documents the project's actual development history — the early S&P-inclusive attempt, and the later, better-performing switch to US VIX. Cells 11/16/18 are the notebook's real, final answer; Cell 4/5 is documented exploratory work.

## S&P 500: Full Test Results (why it was excluded from later notebooks)

Tested three ways on notebook 12's data-driven model, all with proper time-series cross-validation:

| Test | Without S&P | With S&P | Result |
|---|---|---|---|
| Signed return, full dataset (4-fold) | 64.8% | 58.9% | **−6.0 pts** |
| Signed return, 2021-2024 (3-fold) | 73.6% | 71.3% | **−2.3 pts** |
| Absolute move, full dataset (4-fold) | 64.8% | 57.3% | **−7.5 pts** |
| Absolute move, 2021-2024 (3-fold) | 73.6% | 69.0% | **−4.6 pts** |

**Conclusion: every S&P-based feature tested (signed, absolute, and both together) consistently worsens out-of-sample accuracy.** This matches notebook 9's earlier finding that S&P's correlation with NIFTY IV changes sign in 2025, while US VIX's does not. As a result, **dead S&P-loading code has been removed from notebooks 12, 13, 14, and 15** (it was loaded but never used in their actual feature lists — pure leftover scaffolding from early iterations).

## Alternative Model Comparison (tested, not assumed)

Four model families compared on notebook 12's task, same data, same time-series CV:

**Full dataset (2021-2025), 4-fold CV:**
| Model | Fold 1 | Fold 2 | Fold 3 | Fold 4 | Average |
|---|---|---|---|---|---|
| LogisticRegression | 0.594 | 0.625 | 0.625 | 0.469 | 0.578 |
| **GradientBoosting (current)** | 0.844 | 0.656 | 0.594 | 0.500 | **0.648** |
| RandomForest | 0.812 | 0.625 | 0.531 | 0.469 | 0.609 |
| XGBoost | 0.531 | 0.531 | 0.594 | 0.469 | 0.531 |

**2021-2024 only, 3-fold CV:**
| Model | Fold 1 | Fold 2 | Fold 3 | Average |
|---|---|---|---|---|
| LogisticRegression | 0.621 | 0.414 | 0.655 | 0.563 |
| **GradientBoosting (current)** | 0.828 | 0.655 | 0.724 | **0.736** |
| RandomForest | 0.862 | 0.621 | 0.621 | 0.701 |
| XGBoost | 0.586 | 0.552 | 0.724 | 0.621 |

**Disagree-day model (small sample, n=41), 3-fold CV:**
| Model | Average |
|---|---|
| **LogisticRegression (current)** | **0.700** |
| RandomForest | 0.700 (tied) |
| XGBoost | 0.667 |

**Conclusion: GradientBoosting is genuinely the best model for the main data-driven task** — not an arbitrary choice, confirmed against 3 alternatives on 2 different dataset windows. For the small disagree-day dataset, LogisticRegression ties with RandomForest and both are reasonable; the project's use of LogisticRegression there (favoring simplicity on thin data) is a defensible, non-arbitrary choice.

## Files Removed From Final Deliverables

- `13_PnL_Backtest.ipynb` (options-based) — removed, structurally flawed instrument choice for an IV-direction signal
- `13_PnL_Backtest_Futures.ipynb` — removed, tests price-direction rather than the project's actual IV-direction target
- Dead S&P-loading code stripped from notebooks 12, 14, 15, 16 (loaded but never used in any feature list in those notebooks)

## Final Notebook Set (14 notebooks, verified to run end-to-end)

01, 02, 03, 04, 05, 06, 07, 08, 09, 10, 11, 12, 13


## Update: Notebooks Consolidated (13, 14, 15 → single Notebook 13)

The three separate P&L backtest notebooks (calendar-only vega structure test, train-once yearly test, and rolling 6-month retrain test) were consolidated into a **single final notebook 13**, using the rolling 6-month train/test design (the strongest of the three validation approaches) and testing **both calendar spread and butterfly structures side by side**, each with its own independent Rs 100,000 compounding portfolio per strategy. This removes redundant intermediate notebooks while keeping the most rigorous validation method and broadening structure coverage.
